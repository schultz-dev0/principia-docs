# Principia — Interstellar System Support Audit

**Repo audited:** `/home/schultz/principia` (branch `master`, head `cc6522fc9`)
**Question:** How does Principia handle the five interstellar systems
(Alpha Centauri, Barnard's Star, Tau Ceti, Teegarden's Star, Trappist-1),
interstellar travel, galactic motion, and large-distance floating-point precision?

---

## 1. TL;DR

| Topic | Answer |
|---|---|
| Alpha Centauri | **Not supported.** No model, no data, no code path. |
| Barnard's Star | **Not supported.** Same. |
| Tau Ceti | **Not supported.** Same. |
| Teegarden's Star | **Not supported.** Same. |
| Trappist-1 | **Supported as a *replacement* system**, not as a co-simulated neighbour of Sol. |
| Interstellar travel | **Not modelled.** Architecturally a single `Ephemeris<Barycentric>` per session. |
| Galactic motion / proper motion of stars | **Not modelled.** No code, no constants, no concept. |
| Floating-point precision | All physical scalars are IEEE-754 `double`. Long-term integration error is mitigated by *compensated summation* (`DoublePrecision<T>`), not wider floats. |

The shortest accurate description: **Principia simulates one star system at a
time.** "Supporting Trappist-1" means you can swap out Sol and run Trappist-1
instead — not run both, and certainly not fly between them.

---

## 2. What is actually shipped for each system?

The astronomical data lives in `astronomy/` as protobuf-text and Kopernicus
configs. A directory listing is the audit:

```
astronomy/sol_gravity_model.proto.txt
astronomy/sol_initial_state_jd_2433282_500000000.proto.txt
... (many sol_initial_state_*)
astronomy/trappist_gravity_model.proto.txt
astronomy/trappist_gravity_model_slippist1.cfg
astronomy/trappist_initial_state_jd_2457000_000000000.proto.txt
astronomy/trappist_numerics_blueprint.proto.txt
astronomy/trappist_rss_time_formatter.cfg
```

A whole-tree grep for `alpha[ _-]?centauri|barnard|tau[ _-]?ceti|teegarden|proxima`
returns **zero matches** in any source, config, comment, or proto.txt. Of the
five systems requested, only **Trappist-1** has any presence in the codebase.

### Sol

The "real" solar system: Sun, planets, major moons. Multiple initial-state
snapshots at different Julian dates (e.g. `JD2433282.5`, `JD2451545.0` =
J2000.0). Used both for production (KSP via the RealSolarSystem mod) and as
the basis of every `solar_system_dynamics_test`, `lunar_eclipse_test`,
`mercury_perihelion_test`, etc.

### Trappist-1

Star + seven planets b–h, fitted against transit-timing data:

- `trappist_gravity_model.proto.txt` — masses, radii, J2, axes; the star is
  `0.0890 GM☉`, planets in earth-mass units. Frame is `SKY` (see §4).
- `trappist_initial_state_jd_2457000_000000000.proto.txt` — Keplerian
  elements for each planet, with the star at the root and each planet
  parented to it. Comments record the χ² of the orbital fit
  (358.789721, mean transit Δt = 57.9 s).
- `trappist_gravity_model_slippist1.cfg` — Kopernicus patch for the
  **SLIPPIST-1** KSP mod. This is a `@Kopernicus:AFTER[aSLIPPIST-1]` patch
  that **rewrites Kerbal/Slippist body parameters** (`@Body[Sun]`,
  `@Body[Bravo]`, etc.) so that, when the player has the SLIPPIST-1 KSP mod
  installed, Principia's gravitational parameters override Slippist's stock
  values to be physically self-consistent.

The Trappist-1 deployment is therefore a *Kerbal-side mod replacement*: the
game's planetary system *is* Trappist-1 instead of Kerbol. Principia is not
running Sol and Trappist-1 in parallel; whichever Kopernicus configuration
the player loaded at the start of the save is the only star system in the
ephemeris. See §6 for the architectural reason.

---

## 3. Numeric precision: what does Principia actually use?

### Scalars are `double`

Every physical quantity bottoms out in a single IEEE-754 `double`:

- `quantities/quantities.hpp:83` — `Quantity<D>` stores `double magnitude_ = 0;`
- `geometry/r3_element.hpp` — three scalars (each a `Quantity`, hence `double`).
- `geometry/point.hpp` — affine point storing a vector-space element of the
  same.
- No `long double`, no `__float128`, no boost multiprecision used in the
  hot path. `boost_multiprecision.props` exists in the build files but is
  not consumed by physical state.

### `DoublePrecision<T>` is **compensated summation**, not a wider float

`numerics/double_precision.hpp:58` defines:

```cpp
template<typename T>
struct DoublePrecision final {
  T value;
  Difference<T> error;
};
```

This is the **Higham double-double / TwoSum** trick: `value` is the
nominal result, `error` is the rounding residue from the last
addition, both stored as separate `double`s. The relevant primitives:

- `QuickTwoSum(a, b)` — exact sum when |a| ≥ |b|.
- `TwoProduct(a, b)` — exact product (uses FMA when present).
- `Increment` / `Decrement` — apply a correction without re-renormalising.

Effective precision is roughly **2× the mantissa of `double`**
(~32 decimal digits) for *additions/subtractions* of numbers of similar
magnitude. It does **not** widen the dynamic range; the exponent is still
`double`.

### Where `DoublePrecision` is actually used

The integrators carry it through state propagation:

- `integrators/ordinary_differential_equations.hpp:72-73` — the ODE state
  carries `DoublePrecision<IndependentVariable>` (time) and
  `DirectSum<DoublePrecision<DependentVariable>...>` (positions/momenta).
- `integrators/symplectic_runge_kutta_nyström_integrator_body.hpp:61–72` —
  inner loop updates time and position via `.Increment(...)` so that the
  step Δt and Δx do not lose precision against the accumulated
  `t`, `x`.

This is what makes century-scale integrations of Sol stable: the *step*
and the *running total* are summed compensatedly, so the ULP of the
running total does not eat the step.

### Implication for interstellar distances

A `double` resolves about **1 ULP ≈ 2⁻⁵² × |x|** in absolute terms. Using
`x` measured in metres:

| Distance scale | Magnitude | Position ULP |
|---|---|---|
| Earth–Moon | 3.8 × 10⁸ m | ≈ 8.5 × 10⁻⁸ m (sub-µm) |
| Sol → Pluto aphelion | 7.4 × 10¹² m | ≈ 1.6 mm |
| Sol → Voyager-1 | 2.4 × 10¹³ m | ≈ 5 mm |
| 1 AU | 1.496 × 10¹¹ m | ≈ 33 µm |
| Sol → Proxima Centauri | 4.0 × 10¹⁶ m | **≈ 9 m** |
| Sol → Barnard's Star | 5.6 × 10¹⁶ m | ≈ 12 m |
| Sol → Tau Ceti | 1.1 × 10¹⁷ m | ≈ 25 m |
| Sol → Teegarden | 1.2 × 10¹⁷ m | ≈ 27 m |
| Sol → Trappist-1 | 3.8 × 10¹⁷ m | ≈ 84 m |

So the moment a single coordinate origin is asked to span Sol → another
star, you lose **metres-to-tens-of-metres** of position resolution
relative to that star — fine for cruise navigation, fatal for orbital
mechanics around the destination star (a low orbit needs sub-metre
precision to integrate stably over years).

`DoublePrecision<T>` mitigates the *integration* error, not the
*storage* error: if you store the position of Trappist-1f as a single
`Position` referenced to Sol's barycentre, you have already lost ~80 m
the moment you write it down. The double-double stack would help only
if positions were *stored* as `DoublePrecision<Position>` everywhere,
and they are not — `Plugin::ephemeris_` holds a single
`Ephemeris<Barycentric>` of `MassiveBody`s with ordinary `Position`
state. See §6.

### Time precision

`Instant` is a `Point` over `Time`, where `Time` is a `double` of seconds.
Principia's `Instant` corresponds to TT (Terrestrial Time) and is treated
as identified with TCB (`astronomy/frames.hpp:38–40`). The double-double
trick is applied to time as it advances through the integrator — this is
what lets a 100-year Sol integration not drift into nanoseconds-of-step
quantisation.

---

## 4. Reference frames: ICRS, Barycentric, and the `Sky` frame

Principia distinguishes several frames in `astronomy/frames.hpp`:

- `ICRS` (line 41) — barycentre of the **solar** system, ICRF axes.
- `GCRS` (line 61) — geocentric, ICRS axes.
- `ITRS` (line 95) — Earth-fixed.
- `Sky` (line 138) — *barycentre of an extrasolar system*, with the
  z-axis pointing toward Earth and the xy plane being the plane of the
  sky. **This is an observational frame for transit photometry**, not
  a frame for inter-system simulation.

The KSP plugin uses its own `Barycentric` frame
(`ksp_plugin/frames.hpp:40-43`):

```cpp
using Barycentric = Frame<serialization::Frame::PluginTag,
                          Inertial,
                          Handedness::Right,
                          serialization::Frame::BARYCENTRIC>;
```

Comment at `frames.hpp:39`: *"The barycentric reference frame of the
solar system."* (Singular.)

The Trappist-1 protos are written in `solar_system_frame: SKY`. When
you read this through `physics/solar_system.hpp`, *you get a
`SolarSystem<Sky>`* — the `Sky` frame is just being reused as the
inertial frame for the Trappist-1 ephemeris, with its own barycentre.
Two distinct invocations:

- `SolarSystem<ICRS>` — the production Sol ephemeris.
- `SolarSystem<Sky>` — the Trappist-1 test/optimisation ephemeris.

These are **separate template instantiations**, not two views of one
universe. `astronomy/trappist_dynamics_test.cpp` builds an
`Ephemeris<Sky>` from those Trappist-1 protos and integrates it in
isolation; it has no concept of where Sol is.

---

## 5. Galactic motion

There is no model of galactic motion of any kind. A whole-tree grep for:

```
galactic | galaxy | milky way | parsec | light year | lightyear
Sgr A   | barycentre of (the )?galaxy | proper motion (of stars)
```

returns **zero hits in physics/, ksp_plugin/, astronomy/, integrators/**.
The single `proper motion` reference in `ksp_plugin/planetarium.hpp` is
the geometric "proper motion of a point on the celestial sphere as seen
from a vessel" — a 2-D angular projection for planetarium rendering, not
the astronomical proper motion of a star against the galactic
background.

There is no:

- Constant for the distance to Sgr A* or the galactic centre.
- Catalogue of stellar proper motions or radial velocities.
- Hierarchical frame nesting beyond *one* `Inertial` root.
- ICRS-to-galactic frame transform (the ICRS axes are treated as
  globally inertial; Principia inherits the IAU's convention that ICRF
  is non-rotating *for solar-system purposes* and stops there).

Astronomically the ICRS *is* tied to the rotational state of the Milky
Way (via the quasar grid that defines its axes) but Principia treats it
as a Newtonian inertial frame for a single star and does not propagate
the heliocentre's orbit around Sgr A*.

---

## 6. Architectural reason: one `Ephemeris` per simulation

The plugin owns a single ephemeris:

- `ksp_plugin/plugin.hpp:566` — `std::unique_ptr<Ephemeris<Barycentric>> ephemeris_;`
- `ksp_plugin/plugin.hpp:148-149` — type-aliased accuracy and step parameters
  are bound to that one frame.
- All consumers (`orbit_analyser`, `flight_plan`, `geometric_potential_plotter`,
  `planetarium`) take the same `not_null<Ephemeris<Barycentric>*>`.

`Ephemeris<Frame>` itself (`physics/ephemeris.hpp`) is templated on a
single frame. There is no mechanism to:

1. Hold two ephemerides that share a global frame.
2. Re-express one ephemeris in another's frame at runtime.
3. Apply gravity from bodies in `Ephemeris<A>` onto trajectories in
   `Ephemeris<B>`.

So even if you wrote `Sol_initial_state.proto.txt` and a hypothetical
`alpha_centauri_initial_state.proto.txt` both in `ICRS`, then loaded
both into a single `Ephemeris<ICRS>`, you would hit the precision
ceiling described in §3 (≈9 m position ULP at Proxima distance) and
*also* be making a physical model — interstellar gravitational
coupling — that the project has explicitly never tried to validate.

---

## 7. What "interstellar travel" support would require

In current Principia, a vessel near Trappist-1 in a `Sky`-framed
ephemeris and a vessel near Sol in an `ICRS`-framed ephemeris simply
live in two unrelated simulations. To bridge them you would need, in
roughly increasing order of difficulty:

1. **A unified inertial frame** — pick ICRS, embed both stars' barycentres
   in it. Trappist-1 currently has no ICRS position recorded in the
   repo; you'd need to add one (RA 23h06m29s, Dec −05°02'29", parallax
   80.21 mas → distance 12.47 pc, plus the Gaia DR3 proper-motion and
   radial-velocity vector).
2. **Position storage that survives 10¹⁷-metre offsets.** The cleanest
   answer is *not* widening floats but a **two-level frame nest**:
   `Galactic → StarBarycentric → Vessel`, with `DoublePrecision<Position>`
   only across the top level, and ordinary `Position` near each star.
   Principia's frame template machinery (`geometry/frame.hpp`) does
   *not* currently support nesting; today's `Frame<...>` is a flat tag.
3. **Two ephemerides simulated jointly.** This is mostly an engineering
   exercise — you'd refactor `Ephemeris` to take a list of
   `MassiveBody`s plus a list of *external attractor* trajectories
   evaluated in the same frame. Inter-stellar tidal effects are
   negligible (≈10⁻¹⁵ of local stellar gravity) so the coupling itself
   is trivial; the precision and frame-nesting work is the hard part.
4. **Stellar proper motion.** Either propagate the second star's
   barycentre as a body of its own under galactic-tide forces (almost
   nothing else acts on it over human timescales), or simply advect it
   linearly with its measured space velocity — adequate over millennia.
5. **Galactic-frame coordinates** — only required if you want
   simulations spanning ≫1 kpc or > 10⁵ years. Sol's orbit around Sgr
   A* is ~230 km/s, period ~225 Myr; over a typical KSP timescale
   (decades) it is well below the precision floor of any reasonable
   per-star frame.

None of (1)–(5) exists in the current codebase.

---

## 8. Concrete file:line citations for follow-up

| Topic | Location |
|---|---|
| Sol gravity model | `astronomy/sol_gravity_model.proto.txt` |
| Sol initial states (multiple epochs) | `astronomy/sol_initial_state_jd_*.proto.txt` |
| Trappist-1 gravity model | `astronomy/trappist_gravity_model.proto.txt` |
| Trappist-1 initial state (fitted) | `astronomy/trappist_initial_state_jd_2457000_000000000.proto.txt` |
| Trappist-1 KSP/Kopernicus patch (SLIPPIST-1) | `astronomy/trappist_gravity_model_slippist1.cfg` |
| Trappist-1 dynamics tests / optimisation | `astronomy/trappist_dynamics_test.cpp` |
| Reference frames (ICRS, GCRS, ITRS, Sky) | `astronomy/frames.hpp:41,61,95,138` |
| Plugin's working frame | `ksp_plugin/frames.hpp:40-43` |
| Single-ephemeris architecture | `ksp_plugin/plugin.hpp:148,566` |
| Quantity = single `double` | `quantities/quantities.hpp:83` |
| `DoublePrecision<T>` (compensated sum) | `numerics/double_precision.hpp:58` |
| ODE state uses `DoublePrecision` | `integrators/ordinary_differential_equations.hpp:72-73` |
| Symplectic integrator inner loop | `integrators/symplectic_runge_kutta_nyström_integrator_body.hpp:61-72` |
| TwoSum / TwoProduct primitives | `numerics/double_precision.hpp` (rest of file) |

---

## 9. Bottom line

Principia is, by design and by code, a **single-star-system N-body
solver**. Of the five interstellar systems requested, only Trappist-1
has any data in the repo, and that data is meant for a KSP campaign in
which Trappist-1 *replaces* the Kerbol/Sol system, not for joint
simulation with Sol. There is no frame nesting, no galactic motion, no
multi-ephemeris coupling, and the position storage is a single `double`
per axis — adequate for a star system, not for distances of 10¹⁶ m+
unless the architecture is extended.

If your project needs to model interstellar trajectories with Principia
as a starting point, treat it as **a high-quality library for
*per-star* dynamics** that you would compose with a separate
inter-stellar propagator, rather than as a system that supports
interstellar physics out of the box.
