# ARES
## Astrodynamics Research Environment for Earth–Mars Mission Design

**ARES** is an open-source computational astrodynamics framework for the analysis and visualization of ballistic Earth-to-Mars mission opportunities.

The software couples **NASA/JPL NAIF SPICE ephemerides**, a heliocentric **Lambert boundary-value formulation**, numerical two-body propagation, launch-window search, and interactive scientific visualization within a reproducible Python/FastAPI/React architecture.

ARES is designed as an engineering and research platform rather than a trajectory-animation demo: orbital quantities displayed by the user interface originate from a single validated Python astrodynamics engine.

> **Scientific scope:** heliocentric, impulsive, two-body Earth→Mars transfer analysis using geometric JPL ephemeris states.

> **Validation status:** ARES has been compared against published NASA Earth-to-Mars mission-design results. It is **not NASA-certified, NASA-validated, or endorsed by NASA/JPL**.

---

## Research Motivation

Preliminary interplanetary mission design requires the coupling of planetary ephemerides, boundary-value trajectory solutions, departure-energy analysis, arrival conditions, and launch-window exploration.

ARES investigates the following computational problem:

> Given an Earth departure epoch \(t_0\) and Mars arrival epoch \(t_1\), what heliocentric ballistic trajectory connects the corresponding planetary ephemeris states, and what departure and arrival energy conditions result?

The framework extends this calculation over a two-dimensional departure/arrival epoch space to construct Earth-to-Mars launch-window maps.

The project was developed around four principles:

1. **Authoritative ephemerides** rather than circular-orbit approximations.
2. **Traceable physical assumptions** and explicit reference-frame conventions.
3. **Independent numerical verification** of solved trajectories.
4. **Separation of scientific computation from visualization.**

---

# Scientific Architecture

```text
                  NASA/JPL NAIF
                  SPICE / DE440
                       │
                       ▼
              Planetary Ephemerides
                       │
              rE(t0), vE(t0)
              rM(t1), vM(t1)
                       │
                       ▼
              Lambert Boundary Value
                    Problem
                       │
                 v1         v2
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
        Departure            Arrival
        Energetics           Conditions
             │                   │
             ▼                   ▼
        Earth v∞             Mars v∞
             │
             ▼
             C3
                       │
                       ▼
             Numerical Propagation
                DOP853 verification
                       │
                       ▼
              Launch-Window Search
                       │
                       ▼
                Porkchop Map
                       │
                       ▼
                    FastAPI
                       │
                       ▼
              React / TypeScript
                       │
                       ▼
             Scientific Interface
```

The scientific source of truth is:

```text
src/ares/
```

The FastAPI layer performs orchestration and serialization. The React/TypeScript frontend does **not** independently evaluate orbital mechanics.

---

# Mathematical Formulation

## 1. Planetary states

At departure epoch \(t_0\), ARES obtains the heliocentric Earth state

\[
\mathbf{x}_E(t_0)
=
\begin{bmatrix}
\mathbf{r}_E(t_0) \\
\mathbf{v}_E(t_0)
\end{bmatrix}
\]

and at arrival epoch \(t_1\), the heliocentric Mars-system state

\[
\mathbf{x}_M(t_1)
=
\begin{bmatrix}
\mathbf{r}_M(t_1) \\
\mathbf{v}_M(t_1)
\end{bmatrix}.
\]

These states are obtained from NASA/JPL NAIF SPICE kernels rather than analytical circular-orbit approximations.

The transfer time is

\[
\Delta t = t_1-t_0.
\]

---

## 2. Lambert boundary-value problem

ARES solves the heliocentric Lambert problem

\[
\mathcal{L}
\left(
\mathbf{r}_E(t_0),
\mathbf{r}_M(t_1),
\Delta t,
\mu_\odot
\right)
\rightarrow
\left(
\mathbf{v}_1,
\mathbf{v}_2
\right)
\]

where

- \(\mathbf{v}_1\) is the heliocentric transfer velocity at Earth departure,
- \(\mathbf{v}_2\) is the heliocentric transfer velocity at Mars arrival,
- \(\mu_\odot\) is the solar gravitational parameter.

The current implementation uses the **Izzo (2015)** Lambert formulation through `lamberthub`, with

- zero complete revolutions (\(M=0\)),
- prograde transfer geometry.

ARES does not implement a separate Lambert approximation in the frontend.

---

## 3. Hyperbolic excess velocity

Earth-relative departure hyperbolic excess velocity is

\[
\mathbf{v}_{\infty,E}
=
\mathbf{v}_1-\mathbf{v}_E(t_0).
\]

Mars-relative arrival hyperbolic excess velocity is

\[
\mathbf{v}_{\infty,M}
=
\mathbf{v}_2-\mathbf{v}_M(t_1).
\]

Their magnitudes are reported in km/s.

Importantly,

\[
v_\infty \neq \Delta v.
\]

ARES therefore does **not** label either quantity as launch or insertion \(\Delta v\).

---

## 4. Characteristic energy

Departure characteristic energy is calculated as

\[
C_3
=
\left\|
\mathbf{v}_{\infty,E}
\right\|^2.
\]

Units are

\[
\mathrm{km^2/s^2}.
\]

\(C_3\) is a launch-energy metric and is **not equivalent to launch-pad \(\Delta v\)**.

---

# Ephemeris and Reference-Frame Convention

ARES uses the following production convention:

| Parameter | Convention |
|---|---|
| Earth | NAIF **399** — Earth body center |
| Mars target | NAIF **4** — Mars system barycenter |
| Observer | NAIF **10** — Sun |
| Reference frame | **J2000** |
| Aberration correction | **NONE** |
| Position unit | km |
| Velocity unit | km/s |
| Internal time interval | s |
| Solar \( \mu \) | 132712440041.279419 km³/s² |

States are therefore **heliocentric geometric states**.

Earth-Moon barycenter ID 3 is deliberately not used for the Earth-relative departure calculation.

### Mars barycenter audit

The production DE440s kernel does not provide Mars body-center target 499 for the required calculation, so ARES uses Mars-system barycenter 4.

An independent reference-body audit using a Mars satellite SPK compared targets 4 and 499.

For the 2026–2027 reference opportunity, the resulting difference in calculated \(C_3\) was approximately

\[
1.24\times10^{-11}\ \mathrm{km^2/s^2},
\]

which is negligible relative to the fidelity of the present two-body mission model.

The convention is documented rather than silently treating NAIF 4 as the Mars body center.

See [`docs/PHYSICS.md`](docs/PHYSICS.md) for the complete model definition.

---

# Numerical Trajectory Verification

Obtaining a Lambert solution is not treated as sufficient verification.

ARES independently propagates the solved departure state

\[
\left[
\mathbf{r}_E(t_0),
\mathbf{v}_1
\right]
\]

under

\[
\ddot{\mathbf r}
=
-\frac{\mu_\odot}{r^3}\mathbf r
\]

from \(t_0\) to \(t_1\).

Numerical integration uses **DOP853**.

The propagated terminal position is compared against the Lambert target:

\[
\Delta\mathbf r
=
\mathbf r_{\mathrm{prop}}(t_1)
-
\mathbf r_M(t_1).
\]

ARES requires the endpoint error to remain within the project's documented numerical tolerance.

The same propagation is used to generate trajectory visualization samples. Consequently, the displayed transfer is a propagated physical trajectory—not a decorative Bézier curve or graphical spline.

---

# Launch-Window Analysis

A single Lambert solution answers:

> What trajectory connects Earth and Mars for these two dates?

Mission design requires the broader question:

> Which combinations of departure and arrival epoch produce useful transfer conditions?

ARES evaluates a two-dimensional grid

\[
(t_{0,i},t_{1,j})
\]

and solves the corresponding Lambert problem for every physically admissible pair.

For each successful cell, ARES records

\[
\left\{
\Delta t,\,
C_3,\,
v_{\infty,E},\,
v_{\infty,M}
\right\}.
\]

Invalid or excluded cells are represented explicitly as invalid/NaN values rather than being assigned zero.

Default exploratory domain:

| Parameter | Default |
|---|---:|
| Departure interval | 2026-07-01 → 2027-03-01 |
| Arrival interval | 2027-03-01 → 2028-06-01 |
| Grid spacing | 5 days |
| Minimum TOF | 100 days |
| Maximum TOF | 500 days |

The default grid evaluates approximately 4,650 departure/arrival combinations.

Repeated Earth and Mars ephemeris states are cached before Lambert evaluation.

---

# Porkchop Plot

ARES converts the launch-window grid into an interactive porkchop representation.

The primary visualization displays characteristic energy

\[
C_3(t_0,t_1)
\]

across departure and arrival dates.

Interactive inspection exposes:

- departure epoch,
- arrival epoch,
- time of flight,
- \(C_3\),
- Earth \(v_\infty\),
- Mars \(v_\infty\).

Candidate extraction currently reports objective-specific extrema such as

- minimum \(C_3\),
- minimum Mars arrival \(v_\infty\),
- shortest admissible time of flight.

ARES deliberately does **not** call any one of these the "best trajectory," because practical mission optimization is multi-objective.

---

# External Validation

ARES has been compared against the **NASA Interplanetary Mission Design Handbook: Earth-to-Mars Mission Opportunities 2026 to 2045**, NASA/TM-2010-216764.

The handbook reports a Type-II ballistic minimum-energy opportunity near

**Earth departure:** 31 October 2026  
**Mars arrival:** 19 August 2027

Comparison:

| Quantity | NASA/MIDAS | ARES |
|---|---:|---:|
| \(C_3\) | 9.144 km²/s² | 9.1841315 km²/s² |
| Mars \(v_\infty\) | ~2.729 km/s | 2.728454 km/s |

For the ARES solution:

\[
v_{\infty,E}
\approx
3.030533\ \mathrm{km/s}
\]

and

\[
C_3
\approx
9.184132\ \mathrm{km^2/s^2}.
\]

The \(C_3\) difference relative to the published NASA value is approximately

\[
0.0401\ \mathrm{km^2/s^2}
\]

or roughly **0.44%**.

This comparison should **not** be interpreted as an expectation of exact numerical equality.

The published MIDAS result and ARES use different trajectory-design formulations and numerical assumptions. ARES currently evaluates a heliocentric two-body Lambert solution using DE440 geometric states and has not been tuned to reproduce the NASA table.

### Validation status

| Validation layer | Status |
|---|---|
| SPICE state sanity tests | PASS |
| Lambert numerical tests | PASS |
| Two-body endpoint propagation | PASS |
| Stage-2 ↔ porkchop consistency | PASS |
| Python engine ↔ API regression | PASS |
| Full-stack reference-case regression | PASS |
| NASA opportunity-region comparison | PASS / consistent |
| JPL Horizons state-vector comparison | **PENDING** |
| Independent GMAT trajectory comparison | **PENDING** |

ARES is **not NASA-certified** and the use of NASA/JPL data does not imply endorsement.

Detailed evidence is maintained in [`docs/VALIDATION.md`](docs/VALIDATION.md).

---

# Reference Mission

A reproducible reference case is retained as a regression test.

### Input

```text
Departure : 2026-10-31 00:00:00 UTC
Arrival   : 2027-08-19 00:00:00 UTC
```

### ARES result

```text
Time of flight : ~292 days
Earth v∞       : 3.030533204821 km/s
Mars v∞        : 2.728453965898 km/s
C3             : 9.184131505524 km²/s²
```

These values originate from the Python engine and are propagated through

```text
Python physics
      ↓
FastAPI
      ↓
React UI
```

They are not hard-coded frontend values.

---

# Verification and Testing

The repository includes automated tests covering

- UTC/ET handling,
- SPICE ephemerides,
- Lambert calculations,
- energetics,
- numerical propagation,
- launch-window grids,
- candidate extraction,
- API serialization,
- invalid-input handling,
- physics/API numerical regression.

Current release verification:

```text
Python / API tests : 67 passed
TypeScript         : PASS
Frontend build     : PASS
```

Run the scientific/backend test suite with

```bash
python -m pytest
```

Frontend verification:

```bash
cd frontend
npx tsc -b
npm run build
```

A critical invariant is maintained:

\[
C_{3,\mathrm{API}}
\approx
C_{3,\mathrm{physics}}
\]

to serialization precision.

---

# Software Architecture

```text
src/ares/
│
├── constants.py
├── ephemeris.py
├── lambert.py
├── trajectory.py
├── energetics.py
├── porkchop.py
│
└── api/
    ├── main.py
    ├── models.py
    ├── serialize.py
    └── routes/

frontend/
│
├── src/
│   ├── api/
│   ├── components/
│   ├── pages/
│   └── types/
│
└── package.json

tests/
docs/
scripts/
data/kernels/
outputs/
```

The architectural boundary is intentional:

> **Scientific calculations remain in Python. UI code consumes results but does not reproduce astrodynamics equations.**

---

# Application

## Mission Design

The Mission Design workspace accepts departure and arrival epochs and displays

- time of flight,
- departure \(C_3\),
- Earth \(v_\infty\),
- Mars \(v_\infty\),
- propagated heliocentric transfer geometry.

![ARES Mission Design](docs/screenshots/mission-design.png)

*The displayed trajectory consists of propagated two-body samples in J2000 coordinates rather than a decorative curve.*

---

## Launch-Window Explorer

The launch-window workspace evaluates a configurable Earth/Mars date grid and displays the resulting porkchop map.

![ARES Launch Window](docs/screenshots/launch-window.png)

Selecting a candidate passes the selected epochs back to the trajectory engine and recomputes the corresponding Stage-2 solution.

The application does not reconstruct trajectory states from chart interpolation.

---

## Scientific Transparency

Model assumptions and comparison results are exposed directly in the application.

![ARES Validation](docs/screenshots/validation.png)

This is intentional: model fidelity and limitations are treated as part of the result rather than hidden implementation details.

---

# Reproducibility

## Requirements

- Python 3.11+
- Node.js 20+
- approximately 31 MB download for the DE440s SPK
- Windows 10/11, Linux, or macOS

No API key is required.

---

## Installation

Clone the repository:

```bash
git clone https://github.com/ayushb14/ares-mars-mission-design.git
cd ares-mars-mission-design
```

Create a Python environment.

### Windows

```powershell
py -3.11 -m venv .venv
.venv\Scripts\activate
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
```

### Linux/macOS

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
```

Download the official NAIF kernels:

```bash
python scripts/download_kernels.py
```

The script populates

```text
data/kernels/
```

with the required production kernels.

Large kernel binaries are deliberately not stored in Git.

---

# Running ARES

Start the backend:

```bash
python -m ares.api
```

API:

```text
http://127.0.0.1:8765
```

Interactive OpenAPI documentation:

```text
http://127.0.0.1:8765/docs
```

In a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Open:

```text
http://127.0.0.1:4177
```

---

# Command-Line Research Workflow

ARES can also be used without the web interface.

### Ephemeris

```bash
python -m ares.ephemeris --date "2026-10-31T00:00:00"
```

### Single Earth→Mars transfer

```bash
python -m ares.trajectory \
  --departure "2026-10-31T00:00:00" \
  --arrival "2027-08-19T00:00:00"
```

### Launch-window analysis

```bash
python -m ares.porkchop \
  --output outputs/earth_mars_2026_porkchop
```

This makes the computational engine usable independently of the graphical interface.

---

# Data Provenance

ARES relies on official NASA/JPL NAIF data.

Production kernels include:

| Kernel | Function |
|---|---|
| `naif0012.tls` / `.pc` | UTC → ET / leap-second handling |
| `de440s.bsp` | JPL DE440 planetary ephemerides |

The repository does not re-license these kernels.

The kernel-download utility retrieves them separately from the official NAIF distribution.

---

# Current Model Limitations

ARES v0.1 should be interpreted as a **preliminary interplanetary mission-design tool**, not a high-fidelity operational flight-dynamics system.

The current model excludes:

- n-body cruise perturbations,
- planetary perturbations during transfer,
- solar-radiation pressure,
- deep-space maneuvers,
- finite-duration propulsion,
- launch-site geometry,
- launch-vehicle performance,
- Earth parking-orbit departure \(\Delta v\),
- Mars orbit insertion \(\Delta v\),
- atmospheric entry,
- entry/descent/landing,
- spacecraft mass and propellant budgets,
- communications geometry,
- surface operations.

Consequently, ARES currently answers preliminary questions such as

> "What ballistic Earth-to-Mars transfer connects these epochs?"

and

> "Where is the low-energy launch opportunity?"

rather than

> "What exact trajectory should an operational spacecraft fly?"

---

# Research Roadmap

Future work is intentionally fidelity-driven.

### Higher-fidelity trajectory validation

- JPL Horizons state-vector cross-validation
- independent GMAT comparison
- n-body propagation comparison
- trajectory correction maneuver modeling

### Departure and arrival mechanics

- Earth parking-orbit departure
- launch \(\Delta v\)
- launch-vehicle C3 constraints
- Mars orbit insertion
- B-plane targeting

### Mars arrival

- atmospheric entry
- entry heating
- aerodynamic deceleration
- parachute/powered descent
- EDL simulation

### Surface mission analysis

- MOLA topography
- landing-site constraints
- illumination
- communications
- power/resource analysis

These extensions should be added only after maintaining independent validation of the underlying physics.

---

# References

1. Park, R. S., Folkner, W. M., Williams, J. G., & Boggs, D. H. (2021). *The JPL Planetary and Lunar Ephemerides DE440 and DE441*. **The Astronomical Journal, 161**, 105. https://doi.org/10.3847/1538-3881/abd414

2. Izzo, D. (2015). *Revisiting Lambert's problem*. **Celestial Mechanics and Dynamical Astronomy, 121**, 1–15. https://doi.org/10.1007/s10569-014-9587-y

3. Burke, L. M., Falck, R. D., & McGuire, M. L. (2010). *Interplanetary Mission Design Handbook: Earth-to-Mars Mission Opportunities 2026 to 2045*. NASA/TM-2010-216764.

4. NASA/JPL Navigation and Ancillary Information Facility. *SPICE Toolkit and Generic Kernels*.

5. International Astronomical Union. *IAU 2012 Resolution B2*.

Complete source provenance and URLs are maintained in [`docs/SOURCES.md`](docs/SOURCES.md).

---

# Citation

If ARES is used in academic work, please cite the repository and the underlying scientific references listed above.

A formal `CITATION.cff` file is planned for a future release.

---

# License

ARES source code is released under the **MIT License**. See [`LICENSE`](LICENSE).

NASA/JPL NAIF kernels remain data distributed by their respective source organizations. They are downloaded separately and are not re-licensed by this repository.

---

## Disclaimer

ARES is research and educational mission-design software. It is not intended for operational spacecraft navigation, flight certification, or safety-critical mission operations.
