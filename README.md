# ARES

**Mars Mission Design & Systems Simulator**

A local Earth→Mars mission-design tool. Planetary states come from official
NASA/JPL NAIF SPICE kernels. Transfers are a heliocentric two-body Lambert
model. A FastAPI service and a React workspace present that engine; they do
not re-implement orbital equations.

**Source of physical truth:** `src/ares/` (Python).

Compared against published NASA mission-design data. ARES is **not**
NASA-certified.

```text
NASA/JPL SPICE (DE440s, LSK)
        ↓
ARES Python physics engine
        ↓
FastAPI  (serialization only)
        ↓
React / TypeScript  (presentation only)
        ↓
Plotly visualization
```

## Overview

ARES computes Earth→Mars ballistic transfers from DE440 geometric states,
reports Earth/Mars hyperbolic excess velocity and characteristic energy
\(C_3\), and maps a launch-window porkchop. The desktop UI is an engineering
workspace around that validated Python core.

Default comparison dates (editable, not an ARES-claimed optimum):

- Earth departure: **2026-10-31** UTC
- Mars arrival: **2027-08-19** UTC

Those civil dates are the NASA/TM-2010-216764 Table 2 Type II ballistic
energy-minimum pair.

## Features

- UTC → ET via NAIF leapseconds (`str2et`)
- Heliocentric geometric states: Earth **NAIF 399**, Mars system barycenter
  **NAIF 4**, observer Sun **NAIF 10**, frame **J2000**, aberration **NONE**
- Earth→Mars Lambert transfer (Izzo 2015, \(M=0\), prograde)
- \(C_3 = |v_{\infty,\mathrm{Earth}}|^2\), Earth \(v_\infty\), Mars \(v_\infty\)
- Two-body DOP853 endpoint check and physically sampled transfer points
- Porkchop / launch-window grid (same Lambert path as a single transfer)
- Local FastAPI + React mission-design application
- No API keys for core local use

Not implemented: n-body cruise, DSM, finite burns, launch-vehicle ΔV,
MOI ΔV, EDL, landing, habitat, Cesium, GMAT.

## Screenshots

![Mission Design — Lambert transfer](docs/screenshots/mission-design.png)

*Mission Design. Transfer curve is a two-body DOP853 sample of the Lambert
arc (J2000, AU, equal-axis scale), not a decorative spline.*

![Launch Window — C3 porkchop](docs/screenshots/launch-window.png)

*Launch Window. Color is \(C_3\) [km²/s²] from the Stage 3 engine. Invalid
cells are blank. Display scale is clipped at 50 km²/s².*

![Validation](docs/screenshots/validation.png)

*Validation. Live ARES values next to the published NASA Table 2 C3.*

## Architecture

```text
NASA/JPL SPICE
        ↓
ARES Python Physics Engine   src/ares/
        ↓
FastAPI                      src/ares/api/     (orchestrate + serialize)
        ↓
React / TypeScript           frontend/         (presentation)
        ↓
Visualization                Plotly
```

There is one physics implementation. The frontend never solves Lambert, C3,
\(v_\infty\), ephemerides, time conversion, or two-body propagation.

Selecting a porkchop cell copies its departure and arrival dates and
**recomputes Stage 2** through the API. The chart is not interpolated.

## Physics Model

Internal units: **km**, **km/s**, **seconds**. Angles in radians.

| Item | Value |
| --- | --- |
| Earth | NAIF **399** — body center / geocenter, **not** Earth-Moon barycenter 3 |
| Mars target | NAIF **4** — Mars system barycenter, **not** Mars body 499 |
| Observer | NAIF **10** — Sun, heliocentric, **not** solar-system barycenter 0 |
| Frame | **J2000** |
| Aberration | **NONE** (geometric states) |
| \(\mu_\odot\) | 132712440041.279419 km³/s² (DE440 / Park et al. 2021) |
| Lambert | Izzo (2015), \(M=0\), prograde, via `lamberthub` |
| \(C_3\) | \(\lvert v_{\infty,\mathrm{Earth}}\rvert^2\) [km²/s²] — **not** launch ΔV |
| Earth \(v_\infty\) | km/s — **not** pad ΔV |
| Mars \(v_\infty\) | km/s — **not** MOI ΔV |

Details: [`docs/PHYSICS.md`](docs/PHYSICS.md).
Full write-up: [`docs/ARES_TECHNICAL_REPORT.md`](docs/ARES_TECHNICAL_REPORT.md).

## NASA/JPL Data

ARES loads official NAIF generic kernels (not re-licensed by this repo):

| Kernel | Role |
| --- | --- |
| `naif0012.tls` / `naif0012.tls.pc` | Leap seconds (UTC → ET) |
| `de440s.bsp` | DE440 short planetary SPK (~31 MB) |

Download (not stored in git):

```text
python scripts/download_kernels.py
```

The script checks the NAIF-published MD5 of `de440s.bsp`
(`3917ee56769db332790c751e2168843d`). Source:
<https://naif.jpl.nasa.gov/pub/naif/generic_kernels/>.

On Windows, ARES prefers `naif0012.tls.pc` (CRLF).

## Installation

Requirements:

- Python 3.11+ (3.12 is fine)
- Node.js 20+ (UI)
- Network once, to fetch ~31 MB of NAIF kernels
- Windows 10/11, or Linux/macOS

No API keys.

### Windows Setup

1. Install Python 3.11+ from <https://www.python.org/downloads/> and tick
   **Add python.exe to PATH**. Install Node.js LTS from
   <https://nodejs.org/>.
2. Open **PowerShell** in the folder that contains this `README.md`.
3. Create and activate a virtual environment:

```text
py -3.11 -m venv .venv
.venv\Scripts\activate
```

If `py -3.11` is not found, use `py -3` or `python`.

4. Install ARES (and pytest):

```text
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
```

5. Download official NAIF kernels:

```text
python scripts\download_kernels.py
```

This writes `naif0012.tls`, `naif0012.tls.pc`, and `de440s.bsp` into
`data\kernels\`.

### Linux or macOS

```text
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
python scripts/download_kernels.py
```

## Running Backend

Terminal 1 — physics API at `http://127.0.0.1:8765`:

```text
python -m ares.api
```

PowerShell helper (repo root):

```text
.\scripts\run_backend.ps1
```

Interactive OpenAPI: `http://127.0.0.1:8765/docs`

Endpoints: `GET /api/health`, `POST /api/trajectory`, `POST /api/porkchop`,
`GET /api/reference-case`. Porkchop grids larger than **12 000** Lambert
cells are **rejected**, not truncated.

## Running Frontend

Terminal 2 — UI at `http://127.0.0.1:4177` (proxies `/api` to the backend):

```text
cd frontend
npm install
npm run dev
```

PowerShell helper:

```text
.\scripts\run_frontend.ps1
```

Open `http://127.0.0.1:4177`.

## Usage

1. Start backend, then frontend.
2. **Mission Design:** set Earth departure and Mars arrival (UTC dates),
   Calculate. Numbers come from the Python engine through FastAPI.
3. **Launch Window:** set the date grid, Generate. Wait a few seconds for
   the default 5-day 2026/27 window (~4650 cells).
4. Click a valid cell or a listed candidate, then **Open trajectory**.
   That recomputes Stage 2 for those exact dates.
5. **Validation:** model assumptions and the NASA Table 2 comparison.

CLI (optional):

```text
python -m ares.ephemeris --date "2026-10-31T00:00:00"
python -m ares.trajectory --departure "2026-10-31T00:00:00" --arrival "2027-08-19T00:00:00"
python -m ares.porkchop --output outputs\earth_mars_2026_porkchop
```

## Earth → Mars Trajectory

Pipeline in `src/ares/trajectory.py`:

1. SPICE states of Earth (399) at departure and Mars barycenter (4) at arrival.
2. Lambert (Izzo 2015, \(M=0\), prograde) for \(\Delta t = \mathrm{ET}_1 - \mathrm{ET}_0\).
3. \(v_{\infty,\mathrm{E}} = v_1 - v_{\mathrm{Earth}}\),
   \(v_{\infty,\mathrm{M}} = v_2 - v_{\mathrm{Mars}}\),
   \(C_3 = |v_{\infty,\mathrm{E}}|^2\).
4. DOP853 two-body propagation of \((r_{\mathrm{Earth}}, v_1)\) over \(\Delta t\);
   endpoint must match \(r_{\mathrm{Mars}}\) within 1 km (same two-body model).

The UI plot uses those propagated samples (typically 201 points).

## Launch Window / Porkchop Plot

Stage 3 (`src/ares/porkchop.py`) evaluates the **same** Lambert path on a
date grid. Invalid cells are NaN, never 0. Default exploratory bounds:

| Bound | Default |
| --- | --- |
| Departure | 2026-07-01 – 2027-03-01 |
| Arrival | 2027-03-01 – 2028-06-01 |
| Step | 5 days |
| TOF | 100–500 days |

Candidates are grid extrema (minimum \(C_3\), minimum Mars arrival \(v_\infty\),
shortest TOF), not a “best mission” ranking. Shortest-TOF cells often sit on
the 100-day floor with huge \(C_3\).

## Validation

Compared against published NASA mission-design data
(NASA/TM-2010-216764 Table 2). ARES is **not** NASA-certified. Using NAIF
kernels does not mean NASA endorses this software.

NASA/TM-2010-216764 Table 2 Type II ballistic energy minimum
(civil dates 2026-10-31 → 2027-08-19):

| Quantity | NASA (MIDAS, patched-conic) | ARES (two-body Lambert, DE440) |
| --- | --- | --- |
| \(C_3\) | 9.144 km²/s² | ≈ 9.184 km²/s² |

Exact equality is not expected: MIDAS is a patched-conic optimizer with a
two-day search and 0.01 \(C_3\) rounding; ARES is heliocentric two-body
Lambert on DE440 geometric states. ARES is not tuned to that table.

JPL Horizons vector comparison remains **PENDING**. See
[`docs/VALIDATION.md`](docs/VALIDATION.md).

## Testing

```text
python -m pytest
cd frontend
npx tsc -b
npm run build
```

API tests require the same kernels. `POST /api/trajectory` \(C_3\) must match
`compute_earth_mars_transfer(...).c3` within serialization tolerance.

## Current Limitations

- Two-body Sun; no planetary perturbations, SRP, DSM, or finite burns
- Mars endpoint is system barycenter (4), not body 499
- No launch-vehicle performance, parking-orbit ΔV, or MOI ΔV
- No atmosphere, EDL, spacecraft mass, propellant, communications, or surface mission
- Desktop engineering UI; mobile is secondary
- Porkchop is a sampled grid; the continuous minimum may lie between nodes

## Project Structure

```text
src/ares/           physics engine (SPICE, Lambert, C3, porkchop)
src/ares/api/       FastAPI facade
frontend/           React + TypeScript + Vite UI
tests/              pytest (physics + API)
docs/               PHYSICS, SOURCES, VALIDATION, technical report
scripts/            kernel download; Windows/Unix run helpers
data/kernels/       download target (binaries not in git)
outputs/            generated porkchop CSV/HTML (gitignored)
```

## Roadmap

Not started in this tree:

- Parking-orbit departure ΔV and launch-vehicle performance
- Mars orbit insertion ΔV
- n-body cruise comparison
- EDL / landing
- Cesium globe, GMAT integration

## References

Full list with URLs: [`docs/SOURCES.md`](docs/SOURCES.md).

- NASA/JPL NAIF generic kernels (LSK, DE440s)
- Park, R. S., Folkner, W. M., Williams, J. G., & Boggs, D. H. (2021).
  The JPL Planetary and Lunar Ephemerides DE440 and DE441.
  *The Astronomical Journal*, 161, 105.
  <https://doi.org/10.3847/1538-3881/abd414>
- Izzo, D. (2015). Revisiting Lambert’s problem.
  *Celestial Mechanics and Dynamical Astronomy*, 121(1), 1–15.
  <https://doi.org/10.1007/s10569-014-9587-y>
- Burke, L. M., Falck, R. D., & McGuire, M. L. (2010).
  *Interplanetary Mission Design Handbook: Earth-to-Mars Mission
  Opportunities 2026 to 2045.* NASA/TM-2010-216764.
  <https://ntrs.nasa.gov/citations/20100037210>
- IAU 2012 Resolution B2 (astronomical unit)

## License

MIT. See [`LICENSE`](LICENSE).

NAIF kernels remain NASA/JPL data. This repository downloads them; it does
not re-license them.
