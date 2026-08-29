---
id: F-scatter-layout-verb
title: "The plugin's own wiki prescribes a jittered hex lattice at canopy-diameter spacing with a fixed seed and threshold-carved corridors, and ships no verb that produces one — so every caller hand-rolls the lattice in python.execute, and getting the jitter distribution wrong is silent"
status: OPEN
severity: Medium
category: feature
tags: [spatial, scatter, layout, hex-grid, jitter, seed, vegetation, foliage, ism, hism, doctrine-without-implementation, pure-function, reproducibility]
---

# Documented doctrine, implemented nowhere

`Docs/wiki-src/level-building.instancing-and-scatter.md:74-79` is unusually specific for a doc page.
Verbatim:

> - **Dart-throwing with a minimum-distance test is the wrong tool.** Rejection sampling averages
>   far above its own minimum spacing, so the stand comes out visibly sparse and clumped. A
>   **jittered hex grid** gives direct control: lay a hex lattice at the target spacing, offset
>   alternate rows by half a step, and jitter each point by roughly ±0.18 of the spacing.
> - **Space by canopy or footprint diameter, not by taste.** Centres 1.0–1.25 canopy-diameters apart
>   reads as a stand; a 950 uu canopy wants roughly 1000–1200 uu spacing.
> - **One global grid, not one grid per zone**, or overlapping zones double-seed their intersection.
> - **Fix the seed.** A named constant seed makes the layout reproducible and makes a diff between
>   two runs meaningful.
> - Vary per-instance uniform scale by ~±15% and yaw fully; keep pitch and roll at zero unless you
>   mean it.
> - Carve corridors by thresholding a smooth field over the same grid rather than by hand-deleting
>   instances, so the carve survives a rebuild.

The same page argues against reaching for PCG to do it: *"PCG is not always the right scatter tool. A
jittered lattice with a noise density mask (below) is easier to reason about and reproduce for an
organic-looking stand; a general PCG graph is a heavier dependency for the same result"* (`:64`).

**No verb produces that lattice.** The `spatial` namespace ships `raycast`, `raycast_screen`,
`measure_distance`, `measure_overlap`, `verify_placement`, `find_clear_placement`,
`place_on_surface`, `place_relative`, `ground_actors`, `verify_grounding`
(`Handlers/Spatial/*.cpp`, ten `REGISTER_RPC_HANDLER` sites). Every one of them answers a question
about *a placement you already chose*, or moves *one* thing. Nothing generates a set.

So the plugin tells you exactly what to build and leaves you to build it in `python.execute`, every
time, which is the shape this project has already paid for twice.

## Why a hand-rolled lattice goes wrong quietly

The doctrine's numbers are the easy part. The distribution is not.

Jitter is specified as *"roughly ±0.18 of the spacing"* — a **continuous** offset. A
reimplementation on this project drew the jitter as a uniform choice over `(-1, 0, 1)` scaled by the
jitter amount. On a hex lattice with half-step row offsets, that maps two thirds of the points onto
one sublattice and one third onto the other, which is not jitter at all: it is a second, coarser
lattice wearing jitter's name. From overhead it reads as banding; from ground level it reads as
rows. **It renders as a plausible scatter, and no metric anyone was computing could distinguish it
from the right one** — spacing statistics, instance count and bounds are all correct.

That failure is the argument for putting this behind a verb rather than a doc paragraph. A wrong
lattice is not caught by any check the caller would think to run; it is caught by a fixed
implementation with a test.

**Provenance, stated plainly:** the two reimplementations are not citable from this tree. Several
vegetation producers under `Docs/` were written and never committed (this project's `CLAUDE.md`
§ "Version control" records that `p_fx*.py` / `p_rebuild.py` are gone and were never in history), so
the jitter defect above is a recorded observation from the research pass, not a diff a reviewer can
open. Treated as motivation, not as evidence for the mechanism claims, which are all from the doc
page and the handler registrations cited above.

## The ask: a pure function, no side effects

    spatial.scatter_layout — returns transforms; places nothing, spawns nothing, touches no level.

Inputs, matching the doctrine one-for-one so the doc becomes the spec:

- `bounds` (the region), `spacing` (uu, from canopy diameter), `jitter` (fraction of spacing,
  default 0.18, **continuous** uniform in `[-jitter, +jitter]` on both axes)
- `seed` (required, or defaulted to a named constant and echoed — reproducibility is the point)
- `pattern` — `hex` (default) or `square`
- optional `scaleRange` (default ±15% uniform) and `randomYaw` (default true), `pitch`/`roll` left
  at zero per the same page's Rotator trap
- optional `exclude` — a list of regions, or a threshold on a supplied field, so corridor carving is
  part of the layout rather than a post-pass of hand deletions

Output: an array of transforms plus the echoed effective `seed`, `spacing`, `jitter` and `count`.

Three properties make this cheap and worth having:

1. **It is a pure function.** No world access, no actors, no assets, no transaction. It can be unit
   tested in `Tests/Spatial/` with no fixture at all, and the tests are the kind that actually catch
   the defect above: assert the two hex sublattices receive equal counts; assert the jitter
   distribution is continuous (no repeated offsets across a large sample); assert byte-identical
   output for the same seed; assert mean nearest-neighbour distance sits near `spacing` rather than
   well above it, which is the dart-throwing failure the doc names.
2. **It composes with what exists.** Feed its transforms to `foliage.add_instances`, or to
   `actor.spawn_batch`, or ground them with `spatial.ground_actors`. It does not need to know which.
3. **It is testable without an editor**, which matters given how much of this domain currently is
   not.

## Distinct from

- **`F-spatial-no-clear-footprint-search`** (IN-REVIEW, Medium) — *"does a WxH footprint fit near
  here, and if not where does it"*. Its verb shipped as `spatial.find_clear_placement`
  (`Handlers/Spatial/MeasureHandler.cpp:638`). That answers a query about **one** placement against
  existing occupancy; this generates **a set** from a rule, with no world query at all. Adjacent and
  complementary — a caller might carve a layout, then use `find_clear_placement` for the hero props
  — but neither implements the other.
- **`B-foliage-paint-does-no-ground-projection`** (OPEN, High) — the same underlying capability seen
  from the other end: placement that respects the ground. Deliberately split, because this is a
  missing layout verb with no side effects and that is a defective placement verb with side effects;
  either could land alone. But they meet at the obvious pipeline — a layout's transforms handed to a
  `paint` that does not project produces exactly the floating vegetation both exist to prevent — so
  whoever takes one should read the other.
- **`F-ism-per-instance-transforms`** (IN-REVIEW, High) — reads and writes transforms on an existing
  scatter. This produces the transforms in the first place. Note for a fixer: its three verbs are
  **not in this checkout** (zero grep hits under `Source/PinWright/Private/Handlers/`); that work is
  on a sibling host.
- **`F-pcg-create-graph-class-parameter`** (OPEN, High) — the heavyweight route to the same visual
  result, and the one this page's `:64` explicitly argues against for a simple stand. Both are worth
  having; they are not substitutes.
- **`E-duplicate-along-spline-undiscoverable`** and `spline.scatter_meshes_along_spline` — 1D
  distribution along a curve. Different problem.

## Not RPC-verified

No RPC was called; the editor was not running. The claim that no such verb exists is from the ten
`REGISTER_RPC_HANDLER` sites in `Handlers/Spatial/` and a grep of every handler registration for
scatter/lattice/distribute vocabulary — a negative established by search, which is the weakest kind
of claim in this ticket and the one most worth re-checking before implementing.

severity rationale: impact=Medium — soft blocker: the capability is reachable today via hand-written `python.execute`, so nothing is impossible, but it costs a re-implementation per caller of an algorithm the plugin's own docs specify to four decimal places, and the failure mode is silent (a wrong jitter distribution renders as a plausible scatter and passes every count, spacing and bounds check anyone would run) × reach=normal — scatter layout is a core level-building task rather than a rare edge path, and the doctrine page exists precisely because callers hit it repeatedly; the argument for High is that "documented doctrine with no implementation" reliably produces divergent hand-rolls, and it is declined because divergence is a quality cost rather than a blocker and the rubric reserves the High band for false-success or hard blockage -> Medium

## History
- `#1-doctrine-without-verb` `OPEN` reporter — No RPC called, editor not running; this is a source-and-docs review. `Docs/wiki-src/level-building.instancing-and-scatter.md:74-79` specifies a jittered hex lattice — half-step row offset, jitter approximately ±0.18 of spacing, spacing at 1.0-1.25 canopy diameters, one global grid, fixed seed, ±15% uniform scale with full random yaw, corridors carved by thresholding a smooth field — and `:64` argues against using PCG for it. No verb produces it: the ten `REGISTER_RPC_HANDLER` sites in `Handlers/Spatial/` (`raycast`, `raycast_screen`, `measure_distance`, `measure_overlap`, `verify_placement`, `find_clear_placement`, `place_on_surface`, `place_relative`, `ground_actors`, `verify_grounding`) all query or move an existing placement, and a grep of every handler registration for scatter/lattice/distribute vocabulary returns nothing. Ask is a pure `spatial.scatter_layout` that returns transforms and has no side effects, so it unit-tests with no fixture. Motivating failure, recorded as an observation rather than a citable diff: a hand-rolled lattice on this project drew jitter as a uniform choice over (-1, 0, 1) instead of a continuous offset, putting two thirds of the points on one hex sublattice — it renders as banding and passes every count, spacing and bounds check. The producers that did it are not in the tree (several vegetation producers under `Docs/` were never committed; see this project's `CLAUDE.md` § Version control), so that half is unverifiable by a reviewer and is flagged as such in the body. Dedup: searched the board for `scatter`, `hex`, `lattice`, `jitter`, `poisson`, `phyllotax`, `layout` and every `F-spatial-*` / `B-ground-*` file. `F-spatial-no-clear-footprint-search` (IN-REVIEW) is the nearest and is genuinely different — its shipped `spatial.find_clear_placement` (`MeasureHandler.cpp:638`) answers an occupancy query about one placement against the existing world; this generates a set from a rule with no world access. `E-duplicate-along-spline-undiscoverable` and `E-get-splines-info-omits-scattered-meshes` are 1D spline distribution. `F-ism-per-instance-transforms` (IN-REVIEW) reads/writes an existing scatter's transforms and is cross-linked, with the note that its verbs are absent from this checkout. Nothing on the board asks for a layout generator.
