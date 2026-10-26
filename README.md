---

# Scientific Model

## Planetary Ephemerides

For a specified Earth departure epoch `t₀` and Mars arrival epoch `t₁`, ARES retrieves heliocentric position and velocity states from NASA/JPL NAIF SPICE ephemerides.

The Earth state at departure is represented as:

```text
x_E(t₀) = [ r_E(t₀), v_E(t₀) ]
```

where:

- `r_E(t₀)` is the heliocentric position of Earth at departure.
- `v_E(t₀)` is the heliocentric velocity of Earth at departure.

The Mars-system state at arrival is represented as:

```text
x_M(t₁) = [ r_M(t₁), v_M(t₁) ]
```

where:

- `r_M(t₁)` is the heliocentric position of the Mars system barycenter at arrival.
- `v_M(t₁)` is its heliocentric velocity.

The transfer time is:

```text
Δt = t₁ - t₀
```

ARES obtains these states from NASA/JPL NAIF SPICE kernels rather than approximating Earth and Mars as circular orbits.

### Internal Units

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

ARES uses the following reference-body and coordinate conventions:

| Parameter | Convention |
|---|---|
| Earth | NAIF 399 — Earth body center |
| Mars target | NAIF 4 — Mars system barycenter |
| Observer | NAIF 10 — Sun |
| Reference frame | J2000 |
| Aberration correction | NONE |
| Planetary ephemeris | JPL DE440s |
| Solar gravitational parameter | 132712440041.279419 km³/s² |

All planetary states used by the Lambert solver are therefore **heliocentric geometric states expressed in the J2000 inertial frame**.

### Why Earth Uses NAIF 399

ARES uses the physical Earth center (`399`) rather than the Earth-Moon barycenter (`3`).

This is important because departure hyperbolic excess velocity is calculated relative to Earth:

```text
v∞,Earth = v_transfer,departure - v_Earth
```

Using the Earth-Moon barycenter would represent a different physical reference point and would alter the calculated departure energy.

---

## Mars Reference-Body Audit

ARES uses NAIF target `4`, which represents the **Mars system barycenter**.

It must not be confused with:

```text
499 = Mars body center
```

The production `de440s.bsp` kernel used by ARES provides the Mars-system barycenter but does not provide the Mars body-center state required for the same calculation.

To determine whether this distinction materially affects the Earth-to-Mars transfer calculation, a separate reference-body audit was performed using an appropriate Mars satellite SPK containing target `499`.

For the reference 2026–2027 trajectory, the difference between the Mars-system barycenter and Mars body center was approximately:

```text
Position difference at arrival:
0.000204 km

Velocity difference at arrival:
3.88 × 10⁻⁸ km/s
```

The resulting characteristic energies were:

```text
Mars target 4:
C3 = 9.184131505524 km²/s²

Mars target 499:
C3 = 9.184131505536 km²/s²
```

Therefore:

```text
ΔC3 ≈ 1.24 × 10⁻¹¹ km²/s²
```

This difference is negligible relative to the fidelity of the current heliocentric two-body model.

ARES therefore retains:

```text
Earth   = NAIF 399
Mars    = NAIF 4
Sun     = NAIF 10
Frame   = J2000
ABCORR  = NONE
```

The Mars target is explicitly described throughout the project as the **Mars system barycenter**, not the Mars body center.

---

# Lambert Transfer Model

ARES determines the heliocentric ballistic transfer by solving Lambert's boundary-value problem.

The inputs are:

```text
Earth position at departure
Mars position at arrival
Transfer time
Solar gravitational parameter
```

Conceptually:

```text
Lambert(
    r_Earth(t₀),
    r_Mars(t₁),
    Δt,
    μ_Sun
)
        ↓
Transfer departure velocity
Transfer arrival velocity
```

The current implementation uses the **Izzo (2015) Lambert algorithm** through `lamberthub`.

Configuration:

```text
Revolutions : M = 0
Direction   : Prograde
Central body: Sun
```

---

# Hyperbolic Excess Velocity

After solving the Lambert problem, ARES compares the spacecraft transfer velocity with the corresponding planetary velocity.

Earth departure hyperbolic excess velocity:

```text
v∞,Earth = v_transfer,departure - v_Earth
```

Mars arrival hyperbolic excess velocity:

```text
v∞,Mars = v_transfer,arrival - v_Mars
```

ARES reports the magnitudes in `km/s`.

These quantities must not be confused with spacecraft maneuver delta-v.

```text
v∞ ≠ Δv
```

For example, Earth departure `v∞` is not the same quantity as launch-pad delta-v.

---

# Characteristic Energy

ARES calculates Earth departure characteristic energy as:

```text
C3 = |v∞,Earth|²
```

with units:

```text
km²/s²
```

`C3` is a standard measure of hyperbolic departure energy.

It is **not** launch delta-v.

---

# Numerical Trajectory Verification

ARES does not assume that a returned Lambert solution is automatically correct.

The solved departure state is independently propagated from Earth departure to Mars arrival under the two-body solar equation of motion.

Conceptually:

```text
Initial state
    |
    |-- Earth position
    |-- Lambert departure velocity
    |
    v
DOP853 Numerical Integration
    |
    v
Predicted arrival position
    |
    v
Compare with Mars target position
```

The governing acceleration model is:

```text
a = -μ_Sun × r / |r|³
```

where:

- `r` is the heliocentric spacecraft position vector.
- `μ_Sun` is the solar gravitational parameter.

The propagated terminal position is compared against the Mars target position obtained from SPICE.

The endpoint error is:

```text
Δr = r_propagated(t₁) - r_Mars(t₁)
```

The trajectory must satisfy the numerical endpoint tolerance defined by the project.

The same propagated trajectory samples are used by the visualization layer.

Therefore, the trajectory displayed in ARES is **not a decorative spline or Bézier curve**.

---
