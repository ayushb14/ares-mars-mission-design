# ARES

## Earth-to-Mars Mission Design and Trajectory Analysis Platform

ARES is an open-source computational astrodynamics platform for the analysis, design, and visualization of ballistic Earth-to-Mars mission opportunities.

The project combines NASA/JPL planetary ephemerides, Lambert trajectory solutions, numerical propagation, launch-window analysis, porkchop plots, and an interactive mission-design interface.

ARES is designed as an engineering and research platform rather than a trajectory-animation demo. All orbital quantities displayed by the application originate from a single Python physics engine.

> **Scientific scope:** Heliocentric, impulsive, two-body Earth-to-Mars transfer analysis using NASA/JPL ephemeris states.

> **Validation status:** ARES has been compared against published NASA Earth-to-Mars mission-design data. ARES is not NASA-certified or endorsed by NASA/JPL.

---

## Overview

Preliminary interplanetary mission design requires several connected calculations:

- planetary ephemeris determination,
- interplanetary trajectory solution,
- departure-energy analysis,
- arrival-condition analysis,
- numerical trajectory verification,
- launch-window exploration.

ARES addresses the following problem:

> Given an Earth departure epoch and Mars arrival epoch, what heliocentric ballistic trajectory connects the corresponding planetary states, and what departure and arrival conditions result?

The same calculation can be repeated across thousands of departure and arrival combinations to identify useful Earth-to-Mars launch opportunities.

The project follows four main principles:

1. Use authoritative NASA/JPL ephemerides instead of simplified circular planetary orbits.
2. Keep physical assumptions, units, reference frames, and target bodies explicit.
3. Numerically verify trajectory solutions rather than trusting solver output alone.
4. Keep scientific calculations in Python and separate them from visualization.

---

## Key Features

- NASA/JPL NAIF SPICE planetary ephemerides
- JPL DE440s planetary data
- UTC to SPICE Ephemeris Time conversion
- Earth-to-Mars Lambert trajectory solution
- Earth departure hyperbolic excess velocity
- Mars arrival hyperbolic excess velocity
- Characteristic energy C3
- DOP853 numerical trajectory propagation
- Lambert endpoint verification
- Earth-Mars launch-window search
- Interactive porkchop plots
- Candidate trajectory extraction
- FastAPI scientific backend
- React and TypeScript frontend
- Plotly scientific visualization
- Automated physics and API testing
- Comparison against published NASA mission-design data
- No external API key required for core operation

---

## System Architecture

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
      |
      v
Trajectory Verification
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

The complete computation path is:

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

For an Earth departure epoch `t0` and Mars arrival epoch `t1`, ARES obtains heliocentric position and velocity states from NASA/JPL NAIF SPICE ephemerides.

The Earth state at departure is represented as:

```text
x_E(t0) = [r_E(t0), v_E(t0)]
```

where:

- `r_E(t0)` is the heliocentric position of Earth at departure.
- `v_E(t0)` is the heliocentric velocity of Earth at departure.

The Mars-system state at arrival is represented as:

```text
x_M(t1) = [r_M(t1), v_M(t1)]
```

where:

- `r_M(t1)` is the heliocentric position of the Mars system barycenter at arrival.
- `v_M(t1)` is its heliocentric velocity.

The transfer duration is:

```text
dt = t1 - t0
```

ARES obtains these states directly from SPICE kernels rather than approximating Earth and Mars as circular orbits.

### Internal Units

| Quantity | Unit |
|---|---|
| Position | km |
| Velocity | km/s |
| Time interval | s |
| Angle | rad |
| Gravitational parameter | km^3/s^2 |
| Characteristic energy | km^2/s^2 |

---

## SPICE Reference Convention

ARES uses the following production convention:

| Parameter | Convention |
|---|---|
| Earth | NAIF 399 - Earth body center |
| Mars target | NAIF 4 - Mars system barycenter |
| Observer | NAIF 10 - Sun |
| Reference frame | J2000 |
| Aberration correction | NONE |
| Planetary ephemeris | JPL DE440s |
| Solar gravitational parameter | 132712440041.279419 km^3/s^2 |

All planetary states supplied to the Lambert solver are therefore heliocentric geometric states expressed in the J2000 inertial frame.

### Why Earth Uses NAIF 399

ARES uses the physical Earth center, NAIF target `399`, rather than the Earth-Moon barycenter, target `3`.

Earth departure hyperbolic excess velocity is calculated relative to Earth:

```text
v_inf_Earth = v_transfer_departure - v_Earth
```

Using the Earth-Moon barycenter would represent a different physical reference point and alter the calculated departure energy.

---

## Mars Reference-Body Audit

ARES uses NAIF target `4`, representing the Mars system barycenter.

It must not be confused with:

```text
499 = Mars body center
```

The production `de440s.bsp` kernel used by ARES provides the Mars-system barycenter but does not provide the Mars body-center state required for the production calculation.

A separate reference-body audit compared targets `4` and `499` using an appropriate Mars satellite SPK.

For the 2026-2027 reference trajectory, the difference was approximately:

```text
Mars position difference at arrival:
0.000204 km

Mars velocity difference at arrival:
3.88e-8 km/s
```

The corresponding characteristic energies were:

```text
Using Mars target 4:
C3 = 9.184131505524 km^2/s^2

Using Mars target 499:
C3 = 9.184131505536 km^2/s^2
```

Therefore:

```text
Difference in C3 = approximately 1.24e-11 km^2/s^2
```

This difference is negligible relative to the fidelity of the current heliocentric two-body model.

ARES therefore retains:

```text
Earth  = NAIF 399
Mars   = NAIF 4
Sun    = NAIF 10
Frame  = J2000
ABCORR = NONE
```

Throughout the project, target `4` is explicitly identified as the Mars system barycenter rather than the Mars body center.

---

# Lambert Transfer Model

ARES determines the heliocentric ballistic trajectory by solving Lambert's boundary-value problem.

The solver receives:

```text
Earth position at departure
Mars position at arrival
Transfer duration
Solar gravitational parameter
```

Conceptually:

```text
Lambert Solver
     |
     |-- r_Earth(t0)
     |-- r_Mars(t1)
     |-- dt
     |-- mu_Sun
     |
     v
Transfer departure velocity
Transfer arrival velocity
```

The current implementation uses the Izzo Lambert algorithm through `lamberthub`.

Current configuration:

```text
Complete revolutions : M = 0
Direction            : Prograde
Central body         : Sun
```

The frontend does not contain a separate Lambert implementation.

---

# Hyperbolic Excess Velocity

After solving the Lambert problem, ARES compares the spacecraft transfer velocity with the corresponding planetary velocity.

Earth departure hyperbolic excess velocity:

```text
v_inf_Earth = v_transfer_departure - v_Earth
```

Mars arrival hyperbolic excess velocity:

```text
v_inf_Mars = v_transfer_arrival - v_Mars
```

ARES reports their magnitudes in `km/s`.

Hyperbolic excess velocity must not be confused with spacecraft maneuver delta-v.

```text
v_inf != delta-v
```

Earth departure `v_inf` is therefore not equivalent to launch-pad delta-v.

Similarly, Mars arrival `v_inf` is not equivalent to Mars orbit-insertion delta-v.

---

# Characteristic Energy

Earth departure characteristic energy is calculated as:

```text
C3 = |v_inf_Earth|^2
```

Units:

```text
km^2/s^2
```

C3 is a standard measure of hyperbolic departure energy.

It is not launch delta-v.

---

# Numerical Trajectory Verification

ARES does not assume that a returned Lambert solution is automatically correct.

The solved departure state is independently propagated from Earth departure to Mars arrival under the two-body solar gravitational model.

The governing acceleration model is:

```text
a = -mu_Sun * r / |r|^3
```

where:

- `r` is the heliocentric spacecraft position vector.
- `mu_Sun` is the solar gravitational parameter.

ARES uses the DOP853 high-order numerical integration method.

The verification process is:

```text
Earth position at departure
        +
Lambert departure velocity
        |
        v
DOP853 propagation
        |
        v
Predicted arrival position
        |
        v
Compare against SPICE Mars target
```

The endpoint error is:

```text
delta_r = r_propagated(t1) - r_Mars(t1)
```

The propagated trajectory must satisfy the numerical endpoint tolerance defined by the project.

The same propagated trajectory samples are used by the visualization layer.

Therefore, the trajectory shown by ARES is not a decorative spline or Bezier curve.

---

# Earth-to-Mars Launch-Window Analysis

A single Lambert calculation answers:

> What ballistic trajectory connects Earth and Mars for these two dates?

Mission design requires a broader question:

> Which departure and arrival dates produce useful transfer conditions?

ARES constructs a two-dimensional grid of departure and arrival epochs.

Conceptually:

```text
Departure dates
      |
      |        Arrival dates
      |             |
      v             v
   [t0_i]  x     [t1_j]
             |
             v
      Lambert solution
             |
             v
       Mission metrics
```

For each valid date pair, ARES calculates:

```text
Time of flight
C3
Earth v_inf
Mars v_inf
```

Invalid or excluded cells are represented explicitly as invalid values rather than zero.

### Default Search Domain

| Parameter | Default |
|---|---|
| Departure interval | 2026-07-01 to 2027-03-01 |
| Arrival interval | 2027-03-01 to 2028-06-01 |
| Grid spacing | 5 days |
| Minimum time of flight | 100 days |
| Maximum time of flight | 500 days |

Repeated Earth and Mars ephemeris states are precomputed and reused across the Lambert grid.

This reduces unnecessary repeated SPICE evaluations.

---

# Porkchop Analysis

ARES converts the launch-window calculations into an interactive porkchop representation.

The primary visualization maps:

```text
C3(departure date, arrival date)
```

Interactive inspection provides:

- departure epoch,
- arrival epoch,
- time of flight,
- C3,
- Earth `v_inf`,
- Mars `v_inf`.

ARES extracts objective-specific candidates including:

- minimum C3 trajectory,
- minimum Mars arrival `v_inf` trajectory,
- shortest admissible time-of-flight trajectory.

ARES deliberately does not call any one of these the "best mission."

Real interplanetary mission design is a multi-objective optimization problem involving energy, flight time, launch capability, arrival conditions, spacecraft constraints, operations, and risk.

---

# Reference Earth-to-Mars Mission

ARES maintains a reproducible reference mission for regression testing.

## Mission Epochs

```text
Earth departure : 2026-10-31 00:00:00 UTC
Mars arrival    : 2027-08-19 00:00:00 UTC
```

## ARES Results

| Quantity | Result |
|---|---:|
| Time of flight | approximately 292 days |
| Earth v_inf | 3.030533204821 km/s |
| Mars v_inf | 2.728453965898 km/s |
| C3 | 9.184131505524 km^2/s^2 |

These values originate from the Python scientific engine.

They pass through:

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

ARES has been compared against the NASA publication:

**Interplanetary Mission Design Handbook: Earth-to-Mars Mission Opportunities 2026 to 2045**

NASA Technical Memorandum:

```text
NASA/TM-2010-216764
```

For the Type-II ballistic opportunity near:

```text
Departure : 2026-10-31
Arrival   : 2027-08-19
```

the comparison is:

| Quantity | NASA / MIDAS | ARES |
|---|---:|---:|
| C3 | 9.144 km^2/s^2 | 9.1841315 km^2/s^2 |
| Mars v_inf | approximately 2.729 km/s | 2.728454 km/s |

The characteristic-energy difference is approximately:

```text
Delta C3 = 0.0401 km^2/s^2
```

or roughly:

```text
0.44 percent
```

Exact equality is not expected.

The published NASA result and ARES use different mission-design formulations and numerical assumptions.

ARES currently evaluates a heliocentric two-body Lambert solution using DE440 geometric states.

ARES has not been tuned to reproduce the NASA table.

The comparison demonstrates agreement in the location and energetic structure of the 2026 Earth-to-Mars opportunity rather than certification of the software.

ARES is not NASA-certified.

Use of NASA/JPL data does not imply NASA/JPL endorsement.

---

# Verification Matrix

| Verification Layer | Status |
|---|---|
| SPICE ephemeris sanity tests | PASS |
| Time-system tests | PASS |
| Reference-body convention tests | PASS |
| Lambert solver tests | PASS |
| Energetics tests | PASS |
| Two-body endpoint propagation | PASS |
| Trajectory / porkchop consistency | PASS |
| Python engine / API regression | PASS |
| Frontend TypeScript check | PASS |
| Frontend production build | PASS |
| Full-stack reference-case regression | PASS |
| NASA opportunity comparison | CONSISTENT |
| JPL Horizons state-vector comparison | PENDING |
| Independent GMAT trajectory comparison | PENDING |

Detailed validation information is maintained in:

[`docs/VALIDATION.md`](docs/VALIDATION.md)

---

# Automated Testing

The repository contains automated tests covering:

- UTC to Ephemeris Time conversion,
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

Current release verification:

```text
Python / API tests : 67 passed
Failures           : 0
TypeScript         : PASS
Frontend build     : PASS
```

Run the Python scientific and API tests with:

```bash
python -m pytest
```

Verify the frontend with:

```bash
cd frontend
npx tsc -b
npm run build
```

A critical architectural invariant is maintained:

```text
API C3 approximately equals Physics Engine C3
```

to floating-point serialization precision.

---

# Mission Design Interface

The Mission Design workspace allows a user to select Earth departure and Mars arrival epochs and calculate the corresponding transfer.

The interface reports:

- time of flight,
- C3,
- Earth `v_inf`,
- Mars `v_inf`,
- heliocentric transfer geometry.

![ARES Mission Design](docs/screenshots/mission-design.png)

The displayed trajectory consists of numerically propagated two-body trajectory samples in the J2000 frame.

---

# Launch-Window Explorer

The Launch Window workspace evaluates configurable departure and arrival ranges and displays the resulting characteristic-energy landscape.

![ARES Launch Window](docs/screenshots/launch-window.png)

A user can select a launch-window candidate and open the corresponding trajectory.

The selected departure and arrival dates are sent back through the API and the detailed trajectory is recomputed by the Python physics engine.

ARES does not reconstruct detailed trajectory states from chart interpolation.

---

# Scientific Transparency

ARES exposes its physical assumptions and validation status directly in the interface.

![ARES Validation](docs/screenshots/validation.png)

Model fidelity is treated as part of the scientific result rather than being hidden behind the graphical interface.

---

# Repository Structure

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
- internet connection for the initial NAIF kernel download

ARES requires no external API key for its core functionality.

---

# Installation

## Clone the Repository

```bash
git clone https://github.com/ayushb14/ares-mars-mission-design.git
cd ares-mars-mission-design
```

## Windows

Create a virtual environment:

```powershell
py -3.11 -m venv .venv
.venv\Scripts\activate
```

If `py -3.11` is unavailable, use an installed Python 3.11+ interpreter.

Install ARES and development dependencies:

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

Backend address:

```text
http://127.0.0.1:8765
```

Interactive OpenAPI documentation:

```text
http://127.0.0.1:8765/docs
```

Current API endpoints:

```text
GET  /api/health
POST /api/trajectory
POST /api/porkchop
GET  /api/reference-case
```

Large porkchop requests exceeding the configured grid-size limit are rejected rather than silently truncated.

---

# Running the Frontend

Open a second terminal:

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

The scientific engine can also operate independently of the web application.

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

Linux / macOS:

```bash
python -m ares.porkchop --output outputs/earth_mars_2026_porkchop
```

---

# NASA/JPL Data

ARES relies on official NASA/JPL NAIF data.

| Kernel | Purpose |
|---|---|
| `naif0012.tls` / `naif0012.tls.pc` | Leap-second and UTC/ET handling |
| `de440s.bsp` | JPL DE440 planetary ephemerides |

Required kernels are downloaded using:

```bash
python scripts/download_kernels.py
```

The repository does not include the large ephemeris binaries in Git.

The project maintains scientific source information in:

[`docs/SOURCES.md`](docs/SOURCES.md)

---

# Current Scientific Limitations

ARES v0.1 is a preliminary interplanetary mission-design environment.

It is not an operational spacecraft flight-dynamics system.

The current model does not include:

- n-body cruise perturbations,
- planetary perturbations during transfer,
- solar-radiation pressure,
- deep-space maneuvers,
- finite-duration burns,
- launch-site geometry,
- launch-vehicle performance,
- Earth parking-orbit departure delta-v,
- Mars orbit-insertion delta-v,
- atmospheric entry,
- entry, descent, and landing,
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

Future development is intended to increase physical fidelity and mission-design capability.

## Higher-Fidelity Validation

- JPL Horizons state-vector comparison
- independent NASA GMAT trajectory comparison
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

## Landing-Site Analysis

- MOLA topography
- terrain constraints
- elevation analysis
- candidate landing regions

## Surface Mission Analysis

- solar illumination
- communications
- surface power
- mission resources

New physical models should be introduced only with corresponding verification, testing, and documented assumptions.

---

# Technical Documentation

Detailed documentation is available in:

- [`docs/PHYSICS.md`](docs/PHYSICS.md)
- [`docs/VALIDATION.md`](docs/VALIDATION.md)
- [`docs/SOURCES.md`](docs/SOURCES.md)
- [`report/ARES_TECHNICAL_REPORT.md`](report/ARES_TECHNICAL_REPORT.md)

---

# References

1. Park, R. S., Folkner, W. M., Williams, J. G., and Boggs, D. H. (2021).  
   **The JPL Planetary and Lunar Ephemerides DE440 and DE441.**  
   *The Astronomical Journal, 161*, 105.  
   https://doi.org/10.3847/1538-3881/abd414

2. Izzo, D. (2015).  
   **Revisiting Lambert's Problem.**  
   *Celestial Mechanics and Dynamical Astronomy, 121*, 1-15.  
   https://doi.org/10.1007/s10569-014-9587-y

3. Burke, L. M., Falck, R. D., and McGuire, M. L. (2010).  
   **Interplanetary Mission Design Handbook: Earth-to-Mars Mission Opportunities 2026 to 2045.**  
   NASA/TM-2010-216764.  
   https://ntrs.nasa.gov/citations/20100037210

4. NASA/JPL Navigation and Ancillary Information Facility.  
   **SPICE Toolkit and Generic Kernels.**  
   https://naif.jpl.nasa.gov/

5. International Astronomical Union.  
   **IAU 2012 Resolution B2.**

---

# License

ARES source code is released under the MIT License.

See [`LICENSE`](LICENSE).

NASA/JPL NAIF kernels remain data distributed by their respective organizations. ARES downloads these separately and does not re-license them.

---

# Disclaimer

ARES is research and educational mission-design software.

It is not intended for operational spacecraft navigation, flight certification, safety-critical mission operations, or real-world spacecraft commanding.
