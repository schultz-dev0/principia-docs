# Theoretical Implementation: Adding Interstellar Systems to Principia

## Approach Comparison

Two options exist for placing foreign star systems in a shared coordinate space:

| Frame origin | Distance to closest foreign star | ULP at that star's position |
|---|---|---|
| Sol (Option 1) | 4.13 × 10¹⁶ m (Alpha Cen) | ~9 m |
| Galactic centre (Option 2) | 2.46 × 10²⁰ m (Sol itself) | ~55 km |

**Option 1 (Sol at origin) is the correct approach.** With the galactic centre
as origin, Sol itself only resolves to ~55 km precision — planetary orbits are
unworkable before you place a single planet. Option 2 is only viable with a
full hierarchical frame system (galactic frame → star barycentre → vessel),
which is an architectural redesign of Principia. Option 1 is a natural
extension of the existing architecture.

---

## Why Option 1 Works

Sol stays at `(0,0,0)`, exactly as today. Other stars are added to the
`SolarSystem<ICRS>` proto as massive bodies at their current ICRS Cartesian
positions, derived from Gaia DR3 data (RA, Dec, parallax → Cartesian, plus
space velocity for proper motion propagation).

Their planets are expressed as Keplerian elements parented to their host star.
The orbital offsets (~10⁹–10¹¹ m from the star) are small enough that the
force calculation subtraction `r_planet - r_star` is well-conditioned.

---

## Precision Hazard: Cancellation in Force Calculations

The one real precision issue with Option 1: when a distant star's planet is
stored in the global Sol-centred frame, computing `r_planet - r_star` subtracts
two numbers of magnitude ~10¹⁶ m to get a result of ~10¹⁰ m. This loses
roughly 6–7 significant digits due to catastrophic cancellation — adequate for
tracking stellar motion but problematic for high-precision planetary orbit
integration around the distant star.

For stellar-level motion (no planets, just tracking where each star is and its
gravitational pull on Sol's system), this is a non-issue. For full planetary
simulation around a remote star, a per-star local frame would eventually be
needed.

---

## Concrete Steps: Adding a Star as a Gravitational Perturber

Using Alpha Centauri AB as the example. No code changes to Principia itself
are required — this is purely data.

### 1. Get ICRS position and velocity from Gaia DR3

Convert from observational parameters to Cartesian ICRS:

- Right ascension, declination, parallax → (x, y, z) in metres
- Proper motion (μ_α*, μ_δ) + radial velocity → (vx, vy, vz) in m/s

Alpha Centauri A reference values:
- Distance: ~1.34 pc = 4.13 × 10¹⁶ m
- Space velocity: ~25 km/s relative to Sol (combination of proper motion and
  radial approach velocity)

### 2. Add bodies to the Sol gravity model proto

In `astronomy/sol_gravity_model.proto.txt`, add new `body` blocks:

```
body {
  name                    : "Alpha Centauri A"
  gravitational_parameter : "1.100 GM☉"
  mean_radius             : "1.2234 R☉"
}
body {
  name                    : "Alpha Centauri B"
  gravitational_parameter : "0.9070 GM☉"
  mean_radius             : "0.8632 R☉"
}
```

### 3. Add initial state to the matching initial state proto

In the corresponding `sol_initial_state_jd_*.proto.txt`, add Cartesian state
vectors for each new body (position in metres, velocity in m/s relative to the
Sol barycentre in ICRS):

```
body {
  name     : "Alpha Centauri A"
  x        : "..."   # ICRS x in m
  y        : "..."   # ICRS y in m
  z        : "..."   # ICRS z in m
  vx       : "..."   # m/s
  vy       : "..."
  vz       : "..."
}
```

### 4. That's it

Principia integrates all bodies in the ephemeris together. Alpha Centauri's
gravitational effect on Sol's planets is physically negligible (~10⁻⁷ of the
Sun's) but the machinery handles it with no further changes.

---

## Five Target Systems: Reference Data Needed

For each system, you need the Gaia DR3 (or best available) values for:
- Parallax → distance in metres
- RA + Dec → ICRS unit vector
- Proper motion (μ_α*, μ_δ) in mas/yr
- Radial velocity in km/s
- Stellar mass(es) in solar masses

| System | Distance | Notes |
|---|---|---|
| Alpha Centauri (A+B+Proxima) | 1.34 pc / 4.13 × 10¹⁶ m | Triple system; A and B orbit each other with ~80 yr period |
| Barnard's Star | 1.83 pc / 5.64 × 10¹⁶ m | Fastest proper motion of any star (10.3″/yr); single red dwarf |
| Tau Ceti | 3.65 pc / 1.13 × 10¹⁷ m | G-type, similar to Sol; candidate planetary system |
| Teegarden's Star | 3.83 pc / 1.18 × 10¹⁷ m | M-type red dwarf; two confirmed Earth-mass planets |
| Trappist-1 | 12.43 pc / 3.83 × 10¹⁷ m | Already modelled as a standalone system in the repo |

---

## Trappist-1 Special Case

Trappist-1 is the farthest of the five (3.83 × 10¹⁷ m), giving a position ULP
of ~84 m. Its seven planets have very tight orbits (innermost at ~0.01 AU =
1.5 × 10⁹ m), so the per-planet positional error from global-frame storage is
still many orders of magnitude below the orbital radius — the gravitational
perturber use case works fine.

If you wanted to *play* in the Trappist-1 system with Principia-level
precision, the existing standalone `Ephemeris<Sky>` approach (replace Sol with
Trappist-1) remains the right answer. The two use cases are:

- **Study Trappist-1's gravitational effect on Sol's system over time** →
  add it as a perturber in the Sol ephemeris (Option 1, this doc).
- **Fly ships around Trappist-1's planets** → use the existing SLIPPIST-1
  replacement campaign.

These are not mutually exclusive in principle but would require architectural
work to unify.
