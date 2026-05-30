# N-body Engines Compared: Principia vs Universe Sandbox

**What this doc adds.** [`interstellar-audit.md`](interstellar-audit.md)
covers *how Principia stores state and frames* (precision, ICRS/Sky,
single-ephemeris architecture, the frame-nesting roadmap in its §7), and
[`theoretical_implementation.md`](theoretical_implementation.md) covers
*how to add a star as a perturber*. Neither looks at the **integration
scheme itself** (what integrator, fixed vs adaptive step), nor at **how
another real-time n-body engine handles the same problems**. This doc
fills those two gaps by comparing Principia against **Universe Sandbox**
— the most prominent real-time interactive n-body simulator — and asks
what, if anything, a custom Principia branch should borrow from it.

The short version up front: on the **large-distance precision** question
the audit already answered, Universe Sandbox is a **cautionary data
point, not a model** — it does not solve the problem, it lives with the
inaccuracy because it is an interactive sandbox. That *reinforces* the
audit's §7 conclusion (nest frames; don't widen floats; don't expect a
single flat frame to give interstellar precision). The genuinely
borrowable idea is narrow and lives in the UX layer (see §4).

---

## ⚠ Source-quality caveat

The audit cites Principia source by `file:line` because Principia is
open source. **Universe Sandbox is closed-source commercial software** —
there is no source to read. Every US claim below is from public
developer statements (dev blog, FAQ, Steam forum posts by team members)
or the team-maintained wiki, tagged:

- **[DEV]** — direct developer statement (blog / FAQ / forum quote)
- **[WIKI]** — team-maintained wiki, not a verbatim dev quote
- **[INFER]** — inferred from observed behavior / design
- **[NO SRC]** — could not be sourced; treat as unknown

Primary US sources: lead n-body dev "Greenleaf" (Thomas Grønneløv) on
the Steam forums
([integrator thread](https://steamcommunity.com/app/230290/discussions/0/520518053429453785/)),
the [FAQ](https://universesandbox.com/faq/), and dev blog posts
(e.g. the 2023 [Gravity Simulation Upgrade](https://universesandbox.com/blog/2023/08/gravity-simulation-upgrade-update-33/)
and 2016 [N-body](https://universesandbox.com/blog/2016/02/n-body-problem/)
posts).

---

## 1. Integration scheme, side by side

| | Principia | Universe Sandbox |
|---|---|---|
| Default celestial integrator | `QUINLAN_TREMAINE_1990_ORDER_12` — 12th-order **symmetric linear multistep**, conjugate-symplectic | **PEFRL** (4th-order symplectic) [WIKI], from a *selectable suite* [DEV] |
| Step control (celestials) | **Fixed step** (10 min for Sol) | **Adaptive**, controlled by a user-set **tolerance in units of length** via step-doubling error estimate [DEV] |
| Other integrators offered | Blanes-Moan SRKN, McLachlan/-Atela SPRK, Quinlan-1999 SLMS | velocity-Verlet, RK2/RK4, RKF, Adams-Bashforth/Moulton 6th, Hermite 5th, Forest-Ruth, Euler [DEV] |
| Adaptive step used for… | **vessel prediction only** — `DORMAND_ELMIKKAWY_PRINCE_1986_RKN_434FM` (embedded RKN 4(3)4, not symplectic) | **everything** |
| Behavior under time-warp / speed-up | Step stays **fixed**; warp just runs more steps. Correctness preserved | Sim-speed slider is an **auto-limited target**; engine slows to hold the error tolerance, and the UI names the single worst-error body capping speed [DEV+WIKI] |
| Relativity | Newtonian + geopotential spherical harmonics; no GR | Newtonian only; no GR (speed-of-light gravity / PN corrections are aspirational) [DEV] |
| Determinism / reversibility | Fixed-step *symmetric* methods → deterministic, time-reversible, **no secular energy drift** | Adaptive + collision RNG → **not reproducible by design** |

**The core split.** Universe Sandbox makes the integrator *adapt* to keep an
interactive sim alive at any speed; Principia *fixes* the integrator so a
long sim stays physically honest under arbitrary warp. Almost every other
difference falls out of that.

**Note the inversion** of the usual "game = less accurate" intuition for
the *celestial backbone*: Principia's default is **12th-order
symplectic**, higher-order and with stronger long-term guarantees than
US's **4th-order** default — because Principia integrates a known,
relatively stable hierarchy, whereas US must survive arbitrary
user-constructed chaos *plus live collisions* in real time, so it picks a
lower-order adaptive scheme with graceful degradation. Different
problems, not "one is better."

---

## 2. The large-distance precision question (cross-ref to the audit)

The audit's §3 establishes the hazard precisely: a single `double`
position referenced to Sol's barycentre loses **≈9 m of resolution at
Proxima distance** (≈84 m at Trappist-1), and `DoublePrecision<T>`
compensated summation mitigates the *integration* error but **not the
storage** error — the ~80 m is lost the moment the position is written
down. The audit's §7 conclusion is that the clean fix is **frame
nesting** (`Galactic → StarBarycentric → Vessel`, double-double only at
the top level), not wider floats.

**How does Universe Sandbox handle the same scale?** This is the useful
external comparison, and the answer is instructive:

- **No floating-origin / origin-rebasing scheme is documented.** [NO SRC]
  — searched the dev blog, FAQ, and forums; found nothing on how US
  represents positions at galactic scale or combats precision loss.
- **No confirmation of float64 positions either.** [NO SRC] — search
  summaries assert "doubles for position" but it could not be traced to
  a dev statement.
- **What *is* on record:** US self-describes galaxy-scale simulations as
  *"only representative and not very accurate"* [DEV, legacy FAQ]. That
  is a fidelity caveat, but it tells you the design stance: at large
  scale, US **accepts degraded accuracy** rather than engineering a
  precision-preserving frame scheme.

**Implication for the branch.** Universe Sandbox does *not* offer a
solution to import here — if anything it confirms the audit's direction.
The most-used real-time n-body engine simply tolerates the inaccuracy at
interstellar/galactic scale because interactivity, not metre-precision,
is its product. A Principia branch that wants to fly vessels around a
remote star at orbital precision **cannot** copy that; it must do the
frame-nesting work the audit's §7 lays out. US is a "what not to expect"
reference, not a blueprint.

---

## 3. What Universe Sandbox has that Principia lacks

These are real US strengths, listed so the branch can decide
explicitly whether any are in scope. Most are downstream of "interactive
sandbox" and do not fit a deterministic, warp-stable KSP simulation.

- **Collisions / mergers / cratering / fragmentation / Roche tidal
  disruption** [DEV+WIKI] — US's signature feature: overlap detection,
  momentum/energy-conserving merge, fragment spawning, and Roche
  break-up computed from the same per-step tidal evaluation. Principia
  has *none* of this (bodies are point masses on continuous
  trajectories). Adding it would be a large feature fundamentally at
  odds with Principia's model — flag as a product decision, do not
  assume.
- **Barnes-Hut tree for many-body** [DEV] — US falls back from direct
  O(N²) to a Barnes-Hut tree (CPU, via Unity DOTS/Burst — **not GPU**
  [DEV]) for thousands of fragments. Only relevant if the branch
  simulates body counts orders of magnitude beyond a handful of stars +
  planets. For the interstellar-perturber use case (tens of bodies),
  direct summation is trivially fine.

---

## 4. The one borrowable idea: a user-facing accuracy/speed dial

Universe Sandbox's best UX idea is portable **without touching the
symplectic celestial integrator**:

- A single **"max positional error" tolerance** the user can set [DEV].
- A readout of **which body is currently the binding constraint** on
  smooth warp [WIKI].

Principia's correctness is invisible to the player today (fixed step, no
dial). A branch could surface a tolerance and a worst-offender readout
in the diagnostic/UX layer.

**Critical caveat:** do **not** implement this as per-step *adaptive*
stepping of the celestial backbone. Principia's entire no-energy-drift
guarantee rests on the *fixed* symmetric step. The clean implementation
is to expose **step-size selection** (e.g. the 10 min ↔ 2.5 min
Richardson pair Principia already validates against) plus the
worst-offender diagnostic — delivering the UX while preserving the
symplectic property. Adopting US's lower-order adaptive scheme for the
celestials would be a *regression* against the long-term-stability goal,
not an upgrade.

---

## 5. Recommendations for a custom Principia branch

1. **Keep the fixed-step symplectic celestial integrator.** It is
   strictly better than US's default for a deterministic, warp-stable,
   curated system. Do not make the celestial step adaptive.
2. **The real interstellar work is frame nesting**, exactly as the
   audit's §7 describes. Universe Sandbox provides no help here and
   confirms the difficulty is unavoidable for a precision sim.
3. **Borrow only the UX dial + worst-offender readout** (§4), and
   implement it as step-size selection, not adaptive stepping.
4. **Treat collisions/mergers (§3) as an explicit product call.** It is
   the single decision that determines whether a large US-derived
   feature is in or out of scope. There is no Principia machinery for it.

---

## Open questions

- **Does the branch want collisions/mergers at all?** Determines whether
  §3's large feature is in scope. Needs a product decision before any
  engine work.
- **US float precision & floating-origin** [NO SRC]. If we ever need to
  cross-check numerically against US, we'd have to confirm float width
  and any origin scheme — currently unknown.
- **PEFRL exact coefficients / order in the current US build** [WIKI
  only]. The post-rewrite default is wiki-stated, not dev-confirmed.

---

## See also

- [`interstellar-audit.md`](interstellar-audit.md) — Principia's actual
  precision, frames, and single-ephemeris architecture (the precision
  facts this doc builds on)
- [`theoretical_implementation.md`](theoretical_implementation.md) —
  adding a star as a perturber (Sol-at-origin, ICRS Cartesian)
