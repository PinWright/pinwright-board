---
id: F-scatter-layout-verb
title: "The plugin's own wiki prescribes a jittered hex lattice at canopy-diameter spacing with a fixed seed and threshold-carved corridors, and ships no verb that produces one — so every caller hand-rolls the lattice in python.execute, and getting the jitter distribution wrong is silent"
status: DONE
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
- `#2-scatter-layout-shipped` `IN-REVIEW` developer — Premise re-checked in source first, since the ticket flagged its own negative as the weakest claim. It holds. `Handlers/Spatial/` now carries **eleven** `REGISTER_RPC_HANDLER` sites, not ten — `spatial.ground_instances` landed since filing — and a sweep of every registered method name in `PinWright*/` for scatter/layout/grid/lattice/distribute/tile/pattern vocabulary returns only `spline.scatter_meshes_along_spline` (1D along a curve), `geometry.array_linear` / `geometry.array_radial` (which MERGE offset copies into a dynamic mesh's own geometry, not transforms), `image.tile`, `render.capture_ortho_tiles`, `authoring.auto_layout` and `structure.configure_grid_size`. **Nothing generates a set of transforms from a rule.** The five verbs that shipped alongside this ticket are all on the other side of the line: `actor.get_instances` reads, `actor.set_instance_transforms` writes, `spatial.ground_instances` seats, `spatial.find_clear_placement` searches occupancy for ONE pose, `foliage.*` places. Each needs transforms handed to it; none produces them. Ticket not already served.
  **Shipped `spatial.scatter_layout`** (`Source/PinWright/Private/Handlers/Spatial/ScatterLayoutHandler.cpp`, new file — nothing in `Handlers/Spatial/` was edited, so no collision with the two agents working `SpatialTraceUtils.*`). Pure function: no world, no actor, no asset, no trace, no transaction, and therefore **no `movedInstances[]`-style undo record at all**, so `B-ism-undo-record-unsafe`'s defect shape cannot recur here. Params `region` (alias `bounds`, `{min,max}` or `{center,radius}` — the same two shapes `find_clear_placement` takes) and `spacing` required; `pattern` hex|square, `jitter` (fraction of spacing, default 0.18, refused above 0.5), `seed` (default 1337, echoed), `scaleRange` `{min,max}` (default 0.85–1.15), `randomYaw` (default true; pitch/roll always zero with no knob), `exclude[]` carve-out regions, `maxPoints` (default 5000, ceiling 100000). Returns `{count, latticePoints, outsideRegion, excluded, region, layout{…every knob echoed…}, transforms[], units, axis}` where each entry is `{location, rotation, scale}` — the shape `foliage.add_instances`, `actor.spawn_batch` and `actor.set_instance_transforms` already consume, so it composes rather than reimplements. `count + outsideRegion + excluded == latticePoints` always.
  Three decisions worth reviewing. **Jitter is drawn as a continuous double** from `FRandomStream::GetFraction()` mapped to `[-jitter, +jitter] x spacing` — the whole point of the ticket. **The lattice is anchored at the world origin and each point's RNG is seeded from `HashCombine(seed, col, row)`**, not from one running stream, so two overlapping regions agree exactly on their intersection; that is the doc's "one global grid, not one grid per zone" bullet, which neither a region-anchored lattice nor a sequential stream can honour. **All four draws (jitterX, jitterY, yaw, scale) happen unconditionally in a fixed order**, so toggling `randomYaw` does not shift the scale sequence. Containment is tested AFTER jitter, so a returned point is inside the named region rather than inside it plus slop. Refusals are typed `INVALID_PARAMS` (registered as `ERR_INVALID_PARAMS`): shapeless region, spacing below 0.01 cm, unknown pattern, jitter > 0.5, `scaleRange` not `0 < min <= max`, a non-object `exclude` entry (refused, never skipped — a silently uncarved corridor still renders as a plausible layout), an over-budget lattice (refused with the computed count, never truncated), and a lattice index that would not survive narrowing to int32 (a region astronomically far from the origin, or a tiny spacing over a huge one — this one is a real overflow path, guarded in double before any cast).
  **Tests: `Source/PinWright/Private/Tests/Spatial/TestScatterLayout.cpp`**, five cases, no fixture and no editor world needed. `PinWright.spatial.scatter_layout.SameSeedIsByteIdentical` (exact equality on all nine numbers per transform across two runs; a different seed must differ; pitch/roll zero, scale uniform inside 0.85–1.15, Z on the region plane). `…JitterIsContinuousAndBounded` — **the case that catches the recorded defect**: recovers each point's lattice cell, asserts every offset is within ±jitter*spacing, asserts ≥90% of the ~460 offsets are DISTINCT at 1e-4 cm resolution (the `(-1, 0, 1)` hand-roll yields exactly three values and passes every other check), asserts mean |offset| is a real fraction of the jitter, asserts both hex row-parities are populated with neither above 60%, and asserts mean nearest-neighbour distance sits in [0.6, 1.05] × spacing rather than well above it — the dart-throwing failure the doc page names. `…HonoursExclusionsAndBounds`, `…OverlappingRegionsAgreeOnTheirIntersection` (the global-grid property), `…TypedRefusals` (seven refusal cases, each asserting the code and, where two refusals differ in the caller's fix, the distinguishing message text). Not compiled and not run — this wave builds and runs the suite after the fact, so the +5 tests are the figure to reconcile against.
  Docs: `Docs/wiki-src/spatial.md` gains a `### spatial.scatter_layout` H3 at the end of the file (no `##` follows it, so the overlay-structure rule holds), and the doctrine page `Docs/wiki-src/level-building.instancing-and-scatter.md` gains one paragraph under "Scatter That Reads As Deliberate" pointing at the verb — without it the page still tells callers to hand-roll the thing this ticket exists to stop them hand-rolling. No overlap with the three sibling foliage tickets: nothing under `Handlers/Environment/` or `foliage.md` was touched.
- `#3-verified-over-34-calls-plus-distribution-measurements` `DONE` tester — **Verified live over 34 calls to `spatial.scatter_layout`. Every invariant `#2` claimed held.** `count + outsideRegion + excluded == latticePoints` held on every call. Two runs at the same seed were **byte-identical across all nine numbers per transform**, and a different seed differed — so `SameSeedIsByteIdentical`'s property is real on the wire, not only in the test. **World-origin anchoring was proven the hard way**: two *overlapping regions of different shapes* agreed **exactly** on their intersection, which a region-anchored lattice or a single sequential stream cannot do — that is `#2`'s "one global grid" decision (`Handlers/Spatial/ScatterLayoutHandler.cpp:30-34`, the ONE GLOBAL GRID bullet; per-point key at `:147-153`, `ScatterLayoutPointStream` hashing `(seed, column, row)`) confirmed behaviourally. Jitter offsets were **continuous, not quantised** — the recorded motivating failure (a hand-roll drawing jitter from `(-1, 0, 1)`) is caught. The feature the ticket asked for exists and behaves as specified; the ticket is closed on that.

  **Distribution measurements from the same pass, recorded here because a tester's entry is where evidence belongs and it would be lost inside a follow-on feature ask.** Nearest-neighbour **bearing** chi-square, normalised to N=1000, critical value **24.7**: one lattice at the shipped `jitter` default of 0.18 scores **888**; two lattices merged, **87**; three merged, **53**; a 13-lattice build with a density mask and a radius cull, **7.0**. The mechanism is geometric, not a bug: jitter below `sqrt(3)/4 = 0.4330` leaves hex rows **disjoint**, so row structure survives — and 0.4330 is **86.6% of the legal range** (`jitter` is refused above 0.5), the default included. Merging fixes the bearing statistic but **destroys the spacing floor**: `nnMin` **10-24 cm on a 700 cm spacing**, with **50.2%** of points inside half the effective spacing — worse than `jitter: 0.5`'s 38.9%. Merging also cannot decorrelate orientation, because every call rows along the same world axes and there is no rotation parameter; rotating lattices offline took a 4-lattice merge from 54 down to **38.6**.

  **This is a finding about the spec, not a failure of the implementation, and must not be read as a returned verification.** `Docs/wiki-src/level-building.instancing-and-scatter.md:74` specifies the jittered hex grid *and argues against the alternative* — "Dart-throwing with a minimum-distance test is the wrong tool. Rejection sampling averages far above its own minimum spacing, so the stand comes out visibly sparse and clumped" — and `PinWright.spatial.scatter_layout.JitterIsContinuousAndBounded` asserts mean nearest-neighbour distance stays within `[0.6, 1.05] x spacing` (`Tests/Spatial/TestScatterLayout.cpp:347-349`) precisely to catch that failure. The doctrine's argument is about the **mean**; these measurements are about the **minimum** and about **orientation**. Opposite tails of the same distribution, not a contradiction: the verb implements the doctrine faithfully and the doctrine did not ask about bearing isotropy.

  **Three residual defects found in the same pass, all filed separately, none a reason to return this ticket.**
  - `E-scatter-layout-maxpoints-clamped-jitter-refused` — `maxPoints` above its ceiling is silently **clamped** while `jitter` above its cap is **refused**, and the docs call both a "ceiling"; `maxPoints: 0` clamps to 1, and the subsequent over-budget error then reports a "1-point budget" the caller never set. **This does NOT contradict `#2`'s "an over-budget lattice (refused with the computed count, never truncated)", and the distinction matters enough to state plainly:** `#2`'s sentence is about the **computed lattice** exceeding the budget, which is indeed refused with the count and never truncated — verified as written. This finding is about the **budget parameter itself** exceeding its own ceiling. Different value, different code path; `#2` is accurate and is not being contradicted.
  - `E-scatter-layout-yaw-range-doc` — yaw is documented 0-360 and returned **normalised to (-180, 180]** by `FTransform`'s quaternion round-trip. Distribution measured and uniform; the values are correct, the documented range is not. Bites only a caller that range-checks the field.
  - `F-scatter-layout-rotation` — the residual ask, filed as a **`latticeRotation` parameter** plus `nnMin` / `nnMean` / fraction-inside-half-spacing statistics echoed in the `layout{}` block, and deliberately **not** as a `minDistance` parameter: a naive minimum-distance ask is what the doctrine line above declines, and correctly so. One real constraint for a fixer, named so it is not discovered late: rotation **breaks the "two overlapping regions agree exactly on their intersection" property** (`ScatterLayoutHandler.cpp:30-34`) unless the rotation is folded into the per-point key at `:147-153`.

  Recorded honestly: none of these follow-on ticket files existed on the board when this entry was written — they are forward references to tickets being filed in the same session. All line numbers above re-derived at HEAD.
