# Interstellar branch — implementation spec

This directory is the **implementation-level continuation of `interstellar-audit.md`**:
where the audit established *what Principia does today* and *what interstellar
support would require* (§7), these docs specify *how to build it*, grounded in a
fresh read of current `master` (head `440310a9` — the audit's `cc6522fc9` line
numbers have drifted, so all citations here were re-verified).

Produced with Claude Code (opus research agents, one per subsystem) for the
NearStars project; sharing here so we're working from the same map.

## Contents
- `impl-spec.md` — master hand-off spec: three workstreams, build order, the two
  cross-cutting decisions. **Start here.**
- `design-draft.md` — the earlier architecture narrative (superseded on the SOI
  question by R3 below; the FP part matches).
- `research/R1-fp-precision.md` — long-distance FP precision, code-level.
- `research/R2-soi-code-map.md` — SOI cutoff + grouped interaction, code map.
- `research/R3-soi-numerics.md` — the numerical-correctness analysis for R2.
- `research/R4-thrust-under-warp.md` — continuous thrust under timewarp.

## Two findings worth flagging up front

1. **`QUINLAN_TREMAINE_1990_ORDER_12` is a *symmetric linear multistep* method,
   not symplectic** (`integrators/methods.hpp:1041`). The no-secular-drift
   property rests on **time-reversibility + smoothness of f**, not symplecticity.
   This reframes the whole SOI-cutoff risk analysis (R3): a *hard* gravity cutoff
   breaks smoothness → per-crossing energy step → random-walk drift, exactly the
   pathology the integrator is chosen to avoid. The fix is a **C² compactly-supported
   taper on the potential** (exactly zero beyond `r_c`, but smooth), plus a Verlet
   neighbour list for the O(N²)→O(N·k) speedup. See R3 for the argument.

2. **The three interstellar workstreams share one data structure.** The per-star
   *subsystem partition* that WS1 (FP precision) builds to localize origins is the
   same object WS2 (SOI grouping) needs to block the pair loop. Build one, static
   to start. See `impl-spec.md §2`.

## NearStars-specific framing (flagged for transparency)

One owner decision diverges from `theoretical_implementation.md`: the **galactic
frame / galactic-scale coordinates are dropped** as unnecessary. The FP fix keeps
`Barycentric` as the sole integration frame and uses only per-star *local origins*
plus `DoublePrecision` inter-origin offsets (WS1). This is the minimal form; the
galactic-origin option is rejected for the reason your own doc gives (Sol itself
resolves to ~55 km at a galactic origin).

Cross-refs to `../nbody-engine-comparison.md` (the US² integrator/UX note) appear
in `impl-spec.md`; that discussion is out of scope for these three workstreams.
