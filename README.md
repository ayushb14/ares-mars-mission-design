# ARES

## Astrodynamics Research Environment for Earth–Mars Mission Design

ARES is an open-source computational astrodynamics framework for the analysis, design, and visualization of ballistic Earth-to-Mars transfer opportunities.

The system combines:

- NASA/JPL NAIF SPICE ephemerides
- JPL DE440 planetary data
- Lambert boundary-value solutions
- hyperbolic excess velocity and characteristic-energy analysis
- numerical two-body trajectory propagation
- Earth–Mars launch-window search
- porkchop-plot generation
- FastAPI scientific backend
- React + TypeScript engineering interface

ARES is designed as an engineering and research platform rather than a trajectory-animation demo.

All orbital quantities displayed by the application originate from a single Python astrodynamics engine.

> **Scientific scope:** Heliocentric, impulsive, two-body Earth-to-Mars transfer analysis using geometric JPL ephemeris states.

> **Validation status:** ARES has been compared against published NASA Earth-to-Mars mission-design data. ARES is not NASA-certified, NASA-validated, or endorsed by NASA/JPL.

---

## Research Motivation

Preliminary interplanetary mission design requires several coupled calculations:

1. planetary ephemeris determination,
2. interplanetary boundary-value trajectory solution,
3. departure-energy analysis,
4. arrival-condition analysis,
5. numerical trajectory verification,
6. launch-window exploration.

ARES investigates the following computational problem:

> Given an Earth departure epoch and Mars arrival epoch, what heliocentric ballistic trajectory connects the corresponding planetary states, and what departure and arrival energy conditions result?

The same calculation can be repeated across thousands of departure/arrival combinations to identify useful Earth-to-Mars launch opportunities.

The project follows four primary principles:

1. **Authoritative ephemerides** rather than simplified circular planetary orbits.
2. **Explicit physical assumptions** and reference-frame conventions.
3. **Independent numerical verification** of solved trajectories.
4. **Strict separation between scientific computation and visualization.**

---

# System Architecture

```text
NASA/JPL NAIF
SPICE / DE440
      |
      v
Planetary Ephemerides
      |
      |-- Earth state at departure
      |-- Mars state at arrival
      |
      v
Lambert Boundary-Value Problem
      |
      |-- Departure transfer velocity
      |-- Arrival transfer velocity
      |
      v
Mission Energetics
      |
      |-- Earth v_inf
      |-- Mars v_inf
      |-- C3
      |
      v
Two-Body Numerical Propagation
DOP853 Endpoint Verification
      |
      v
Launch-Window Search
      |
      v
Porkchop Analysis
      |
      v
FastAPI
      |
      v
React / TypeScript
      |
      v
Scientific Mission-Design Interface
```

The scientific source of truth is:

```text
src/ares/
```

The frontend does not independently calculate orbital mechanics.

The computation path is:

```text
NASA/JPL SPICE
      |
      v
ARES Python Physics Engine
      |
      v
FastAPI
      |
      v
React / TypeScript
      |
      v
Plotly Visualization
```

---

# Scientific Model

## Planetary Ephemerides

At departure epoch $t_0$, ARES obtains the heliocentric Earth state

$$
\mathbf{x}_E(t_0)
=
\begin{bmatrix}
\mathbf{r}_E(t_0) \\
\mathbf{v}_E(t_0)
\end{bmatrix}
$$

and at arrival epoch $t_1$, the heliocentric Mars-system state

$$
\mathbf{x}_M(t_1)
=
\begin{bmatrix}
\mathbf{r}_M(t_1) \\
\mathbf{v}_M(t_1)
\end{bmatrix}.
$$

These states are obtained from NASA/JPL NAIF SPICE kernels rather than analytical circular-orbit approximations.

The transfer duration is

$$
\Delta t = t_1 - t_0.
$$

Internal units are:

| Quantity | Unit |
|---|---|
| Position | km |
| Velocity | km/s |
| Time interval | s |
| Angle | rad |
| Gravitational parameter | km³/s² |
| Characteristic energy | km²/s² |

---

# SPICE Reference Convention

ARES currently uses:

| Parameter | Convention |
|---|---|
| Earth | NAIF 399 — Earth body center |
| Mars target | NAIF 4 — Mars system barycenter |
| Observer | NAIF 10 — Sun |
| Reference frame | J2000 |
| Aberration correction | NONE |
| Ephemeris | JPL DE440s |
| Solar gravitational parameter | 132712440041.279419 km³/s² |

The resulting planetary states are heliocentric geometric states.

Earth-Moon barycenter ID 3 is deliberately not used for the Earth-relative departure calculation.

## Mars Barycenter Audit

The production `de440s.bsp` kernel does not provide the Mars body-center target 499 required for the same calculation, so ARES uses Mars-system barycenter 4.

A separate reference-body audit compared NAIF targets 4 and 499 using an appropriate Mars satellite SPK.

For the 2026–2027 reference trajectory, changing the Mars endpoint from target 4 to 499 changed calculated characteristic energy by approximately:

$$
1.24 \times 10^{-11}\ \text{km}^2/\text{s}^2
$$

which is negligible relative to the fidelity of the current two-body mission model.

ARES therefore retains target 4 while explicitly identifying it as the **Mars system barycenter**, not the Mars body center.

See [`docs/PHYSICS.md`](docs/PHYSICS.md) for additional details.

---

# Lambert Transfer Formulation

ARES solves the heliocentric Lambert boundary-value problem

$$
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
$$

where:

- $\mathbf{v}_1$ is the heliocentric transfer velocity at Earth departure,
- $\mathbf{v}_2$ is the heliocentric transfer velocity at Mars arrival,
- $\mu_\odot$ is the solar gravitational parameter.

The current implementation uses the **Izzo (2015)** Lambert formulation through `lamberthub`.

Current transfer configuration:

- zero complete revolutions,
- $M=0$,
- prograde transfer geometry.

The frontend does not contain an independent Lambert implementation.

---

# Hyperbolic Excess Velocity

The Earth-relative departure hyperbolic excess velocity is

$$
\mathbf{v}_{\infty,E}
=
\mathbf{v}_1
-
\mathbf{v}_E(t_0).
$$

The Mars-relative arrival hyperbolic excess velocity is

$$
\mathbf{v}_{\infty,M}
=
\mathbf{v}_2
-
\mathbf{v}_M(t_1).
$$

ARES reports their magnitudes in km/s.

These quantities are not equivalent to impulsive spacecraft delta-v.

In particular:

$$
v_\infty \neq \Delta v.
$$

Therefore ARES does not label Earth $v_\infty$ as launch-pad delta-v or Mars $v_\infty$ as Mars-orbit-insertion delta-v.

---

# Characteristic Energy

Departure characteristic energy is calculated as

$$
C_3
=
\left\|
\mathbf{v}_{\infty,E}
\right\|^2.
$$

Its units are:

$$
\text{km}^2/\text{s}^2.
$$

$C_3$ is an important launch-energy parameter but is not itself launch delta-v.

---

# Numerical Trajectory Verification

Returning a Lambert solution is not treated as sufficient verification.

ARES independently propagates the solved heliocentric departure state

$$
\left[
\mathbf{r}_E(t_0),
\mathbf{v}_1
\right]
$$

under the two-body solar equation of motion

$$
\ddot{\mathbf r}
=
-\frac{\mu_\odot}{r^3}\mathbf r.
$$

Numerical integration uses the **DOP853** high-order explicit Runge-Kutta method.

The propagated terminal position is compared against the target Mars position:

$$
\Delta \mathbf r
=
\mathbf r_{\mathrm{prop}}(t_1)
-
\mathbf r_M(t_1).
$$

ARES requires the endpoint discrepancy to remain within the project's documented numerical tolerance.

The same propagation model generates the trajectory samples used by the interface.

Therefore the trajectory displayed in ARES is based on propagated physical states rather than a decorative Bézier curve or graphical spline.

---

# Earth-to-Mars Launch-Window Analysis

A single Lambert solution answers:

> What ballistic transfer connects Earth and Mars for these two epochs?

Mission design requires a broader search:

> Which departure and arrival combinations produce useful transfer conditions?

ARES constructs a two-dimensional grid of departure and arrival epochs:

$$
(t_{0,i}, t_{1,j}).
$$

For every physically admissible date pair, the same validated trajectory pipeline is evaluated.

ARES records:

$$
\left\{
\Delta t,\,
C_3,\,
v_{\infty,E},\,
v_{\infty,M}
\right\}.
$$

Invalid or excluded cells are represented as invalid values rather than zero.

Zero therefore remains a physically meaningful numerical value rather than a failure sentinel.

## Default Search Domain

| Parameter | Default |
|---|---|
| Departure interval | 2026-07-01 to 2027-03-01 |
| Arrival interval | 2027-03-01 to 2028-06-01 |
| Grid spacing | 5 days |
| Minimum TOF | 100 days |
| Maximum TOF | 500 days |

Repeated Earth and Mars states are precomputed and reused across the Lambert grid.

This avoids unnecessary repeated SPICE evaluations.

---

# Porkchop Analysis

ARES converts the launch-window grid into an interactive porkchop representation.

The primary map represents:

$$
C_3(t_0,t_1).
$$

Interactive inspection provides:

- departure epoch,
- arrival epoch,
- time of flight,
- characteristic energy,
- Earth hyperbolic excess velocity,
- Mars hyperbolic excess velocity.

ARES also extracts objective-specific candidates including:

- minimum $C_3$ trajectory,
- minimum Mars arrival $v_\infty$ trajectory,
- shortest admissible time-of-flight trajectory.

No candidate is automatically labelled the **best mission** because practical mission design is inherently multi-objective.

---

# Reference Earth-to-Mars Mission

ARES retains a reproducible reference case for regression testing.

## Mission Epochs

```text
Earth departure : 2026-10-31 00:00:00 UTC
Mars arrival    : 2027-08-19 00:00:00 UTC
```

## ARES Results

| Quantity | Result |
|---|---:|
| Time of flight | approximately 292 days |
| Earth $v_\infty$ | 3.030533204821 km/s |
| Mars $v_\infty$ | 2.728453965898 km/s |
| $C_3$ | 9.184131505524 km²/s² |

These values originate from the Python scientific engine.

They are propagated through:

```text
ARES Python Engine
      |
      v
FastAPI
      |
      v
React Interface
```

They are not hard-coded frontend values.

---

# Comparison with Published NASA Mission-Design Data

ARES has been compared against the:

**Interplanetary Mission Design Handbook: Earth-to-Mars Mission Opportunities 2026 to 2045**

NASA/TM-2010-216764.

For the Type-II ballistic opportunity near:

```text
Departure : 2026-10-31
Arrival   : 2027-08-19
```

the comparison is:

| Quantity | NASA/MIDAS | ARES |
|---|---:|---:|
| $C_3$ | 9.144 km²/s² | 9.1841315 km²/s² |
| Mars $v_\infty$ | approximately 2.729 km/s | 2.728454 km/s |

The difference in characteristic energy is approximately:

$$
\Delta C_3
\approx
0.0401\ \text{km}^2/\text{s}^2,
$$

corresponding to roughly **0.44%**.

Exact equality is not expected.

The published NASA result and ARES use different mission-design formulations and numerical assumptions.

ARES currently evaluates a heliocentric two-body Lambert transfer using DE440 geometric states and has **not been tuned to reproduce the NASA result**.

The comparison demonstrates agreement in the location and energetic structure of the 2026 Earth-to-Mars opportunity rather than certification of the software.

---

# Verification Matrix

| Verification Layer | Status |
|---|---|
| SPICE ephemeris sanity tests | PASS |
| Time-system tests | PASS |
| Lambert solver tests | PASS |
| Energetics tests | PASS |
| Two-body endpoint propagation | PASS |
| Stage-2 / porkchop consistency | PASS |
| Python engine / API regression | PASS |
| Frontend production build | PASS |
| Full-stack reference-case regression | PASS |
| NASA opportunity comparison | CONSISTENT |
| JPL Horizons state-vector comparison | PENDING |
| Independent GMAT comparison | PENDING |

ARES is not NASA-certified.

Use of NASA/JPL kernels does not imply NASA/JPL endorsement.

Complete validation information is maintained in:

[`docs/VALIDATION.md`](docs/VALIDATION.md)

---

# Automated Testing

The repository includes automated tests covering:

- UTC to ET conversion,
- SPICE ephemerides,
- reference-body conventions,
- Lambert solutions,
- hyperbolic excess velocity,
- characteristic energy,
- numerical propagation,
- launch-window grids,
- candidate extraction,
- invalid-input handling,
- API serialization,
- API/physics numerical consistency.

Current release status:

```text
Python / API tests : 67 passed
Failures           : 0
TypeScript         : PASS
Frontend build     : PASS
```

Run the scientific/backend suite with:

```bash
python -m pytest
```

Frontend verification:

```bash
cd frontend
npx tsc -b
npm run build
```

A critical architectural invariant is:

$$
C_{3,\mathrm{API}}
\approx
C_{3,\mathrm{physics}}
$$

to serialization precision.

---

# Mission Design Interface

The Mission Design workspace allows a user to select Earth departure and Mars arrival epochs and calculate the corresponding transfer.

The interface reports:

- time of flight,
- characteristic energy,
- Earth $v_\infty$,
- Mars $v_\infty$,
- heliocentric transfer geometry.

![ARES Mission Design](docs/screenshots/mission-design.png)

The displayed transfer consists of numerically propagated two-body trajectory samples in the J2000 frame.

---

# Launch-Window Explorer

The Launch Window workspace evaluates configurable departure and arrival ranges and displays the resulting characteristic-energy landscape.

![ARES Launch Window](docs/screenshots/launch-window.png)

Selecting a launch-window candidate transfers the selected epochs back into the trajectory workflow.

The detailed trajectory is then recomputed through the Python engine.

ARES does not infer detailed trajectory states through chart interpolation.

---

# Scientific Transparency

ARES exposes its assumptions and validation status directly in the interface.

![ARES Validation](docs/screenshots/validation.png)

The purpose is to make model fidelity visible to the user rather than hiding assumptions behind the graphical interface.

---

# Repository Architecture

```text
ares-mars-mission-design/
|
|-- src/
|   `-- ares/
|       |-- constants.py
|       |-- ephemeris.py
|       |-- lambert.py
|       |-- trajectory.py
|       |-- energetics.py
|       |-- porkchop.py
|       |
|       `-- api/
|           |-- main.py
|           |-- models.py
|           |-- serialize.py
|           `-- routes/
|
|-- frontend/
|   |-- src/
|   |   |-- api/
|   |   |-- components/
|   |   |-- pages/
|   |   `-- types/
|   |
|   |-- package.json
|   `-- package-lock.json
|
|-- tests/
|-- docs/
|-- report/
|-- scripts/
|-- data/
|   `-- kernels/
|
|-- outputs/
|-- pyproject.toml
|-- README.md
`-- LICENSE
```

---

# Reproducibility

## Requirements

- Python 3.11 or newer
- Node.js 20 or newer
- Windows 10/11, Linux, or macOS
- internet access once for the required NAIF kernel download

ARES requires no external API key for its core functionality.

---

# Installation

## Clone

```bash
git clone https://github.com/ayushb14/ares-mars-mission-design.git
cd ares-mars-mission-design
```

## Windows

Create the environment:

```powershell
py -3.11 -m venv .venv
.venv\Scripts\activate
```

Install ARES:

```powershell
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
```

Download the required official NAIF kernels:

```powershell
python scripts\download_kernels.py
```

The kernels are stored under:

```text
data\kernels\
```

Large SPICE kernel binaries are deliberately excluded from Git.

---

## Linux / macOS

```bash
python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
python -m pip install -e ".[dev]"

python scripts/download_kernels.py
```

---

# Running the Backend

Start the ARES physics API:

```bash
python -m ares.api
```

Backend:

```text
http://127.0.0.1:8765
```

Interactive OpenAPI documentation:

```text
http://127.0.0.1:8765/docs
```

Current API endpoints include:

```text
GET  /api/health
POST /api/trajectory
POST /api/porkchop
GET  /api/reference-case
```

Porkchop requests exceeding the configured grid-size limit are rejected rather than silently truncated.

---

# Running the Frontend

Open another terminal:

```bash
cd frontend
npm install
npm run dev
```

Open:

```text
http://127.0.0.1:4177
```

The frontend communicates with the local ARES FastAPI service.

---

# Command-Line Research Workflow

The scientific engine can be used independently of the graphical interface.

## Planetary Ephemeris

```bash
python -m ares.ephemeris --date "2026-10-31T00:00:00"
```

## Single Earth-to-Mars Transfer

```bash
python -m ares.trajectory --departure "2026-10-31T00:00:00" --arrival "2027-08-19T00:00:00"
```

## Launch-Window Analysis

Windows:

```powershell
python -m ares.porkchop --output outputs\earth_mars_2026_porkchop
```

Linux/macOS:

```bash
python -m ares.porkchop --output outputs/earth_mars_2026_porkchop
```

---

# Data Provenance

ARES relies on official NASA/JPL NAIF data.

| Kernel | Function |
|---|---|
| `naif0012.tls` / `naif0012.tls.pc` | Leap-second and UTC/ET handling |
| `de440s.bsp` | JPL DE440 planetary ephemerides |

The repository does not redistribute or re-license the large ephemeris kernels.

They are obtained separately using:

```bash
python scripts/download_kernels.py
```

The DE440s download is checked against the expected NAIF-published checksum used by the project.

---

# Current Scientific Limitations

ARES v0.1 is a **preliminary interplanetary mission-design environment**, not an operational flight-dynamics system.

The current model does not include:

- n-body cruise perturbations,
- planetary perturbations during the heliocentric transfer,
- solar-radiation pressure,
- deep-space maneuvers,
- finite-duration burns,
- launch-site geometry,
- launch-vehicle performance,
- Earth parking-orbit departure delta-v,
- Mars orbit insertion delta-v,
- atmospheric entry,
- entry/descent/landing,
- spacecraft mass modeling,
- propellant budgets,
- communications geometry,
- surface operations.

ARES currently addresses questions such as:

> What heliocentric ballistic trajectory connects Earth and Mars for these epochs?

and:

> Where are the lower-energy Earth-to-Mars launch opportunities?

It does not currently answer:

> What exact trajectory should an operational spacecraft fly?

---

# Research Roadmap

Future development is intended to increase model fidelity rather than simply add interface features.

## Higher-Fidelity Validation

- JPL Horizons state-vector comparison
- independent GMAT trajectory comparison
- n-body propagation
- trajectory-correction maneuvers

## Departure and Arrival Mechanics

- Earth parking-orbit departure
- departure delta-v
- launch-vehicle C3 constraints
- Mars orbit insertion
- B-plane targeting

## Mars Arrival

- atmospheric entry
- aerodynamic deceleration
- entry heating
- parachute descent
- powered descent
- EDL simulation

## Surface Mission Analysis

- MOLA topography
- landing-site constraints
- illumination analysis
- communications
- surface power
- resource analysis

New physical models should be introduced only with corresponding verification and documented assumptions.

---

# References

1. Park, R. S., Folkner, W. M., Williams, J. G., & Boggs, D. H. (2021).  
   **The JPL Planetary and Lunar Ephemerides DE440 and DE441.**  
   *The Astronomical Journal, 161*, 105.  
   https://doi.org/10.3847/1538-3881/abd414

2. Izzo, D. (2015).  
   **Revisiting Lambert's problem.**  
   *Celestial Mechanics and Dynamical Astronomy, 121*, 1-15.  
   https://doi.org/10.1007/s10569-014-9587-y

3. Burke, L. M., Falck, R. D., & McGuire, M. L. (2010).  
   **Interplanetary Mission Design Handbook: Earth-to-Mars Mission Opportunities 2026 to 2045.**  
   NASA/TM-2010-216764.  
   https://ntrs.nasa.gov/citations/20100037210

4. NASA/JPL Navigation and Ancillary Information Facility.  
   **SPICE Toolkit and Generic Kernels.**  
   https://naif.jpl.nasa.gov/

5. International Astronomical Union.  
   **IAU 2012 Resolution B2.**

Complete scientific provenance is maintained in:

[`docs/SOURCES.md`](docs/SOURCES.md)

---

# Technical Documentation

Detailed project documentation is available in:

- [`docs/PHYSICS.md`](docs/PHYSICS.md)
- [`docs/VALIDATION.md`](docs/VALIDATION.md)
- [`docs/SOURCES.md`](docs/SOURCES.md)
- [`report/ARES_TECHNICAL_REPORT.md`](report/ARES_TECHNICAL_REPORT.md)

---

# License

ARES source code is released under the **MIT License**.

See [`LICENSE`](LICENSE).

NASA/JPL NAIF kernels remain data distributed by their respective organizations. ARES downloads these separately and does not re-license them.

---

# Disclaimer

ARES is research and educational mission-design software.

It is not intended for operational spacecraft navigation, flight certification, safety-critical mission operations, or real-world spacecraft commanding.
---

## Disclaimer

ARES is research and educational mission-design software. It is not intended for operational spacecraft navigation, flight certification, or safety-critical mission operations.
