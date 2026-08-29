---
id: F-scatter-layout-rotation
title: "`spatial.scatter_layout` pins every lattice to the world axes and reports no spacing statistics, so orientation is inexpressible with any knob it has and the merge workaround trades a bearing artefact for a worse spacing one"
status: OPEN
severity: Medium
category: feature
tags: [spatial, scatter_layout, lattice, rotation, orientation, spacing, nearest-neighbour, statistics, hex-grid, jitter, missing-param]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The lattice is anchored to the world axes by design, and nothing in the verb can turn it

`spatial.scatter_layout` has **no rotation parameter and no minimum-distance guarantee**, and neither
gap is expressible with the knobs it has.

Complete declared parameter list, `Handlers/Spatial/ScatterLayoutHandler.cpp:172-216`:
`region`/`bounds` (`:173-177`), `spacing` (`:178-181`), `pattern` (`:182-186`), `jitter` (`:187-192`),
`seed` (`:193-196`), `scaleRange` (`:197-200`), `randomYaw` (`:201-205`), `exclude` (`:206-210`),
`maxPoints` (`:211-215`). No rotation. No `minDistance`. The dispatcher rejects undeclared top-level
keys, so there is no undocumented escape hatch to try.

`randomYaw` is not the missing knob: it rotates each *mesh* about its own origin (`:201-205`, pitch
and roll always zero). Nothing rotates the *lattice*.

## Why no knob reaches it

**Orientation is welded to the world axes by the arithmetic.** The anchoring is deliberate and
documented at `:30-34`: *"ONE GLOBAL GRID. The lattice is anchored at the WORLD ORIGIN rather than at
the region … Two overlapping regions therefore agree exactly on their intersection instead of
double-seeding it."* It is implemented at `:325-336` (`ColStep`/`RowStep` and the four
`Floor`/`Ceil` bounds solved against world X and Y), with the hex half-step at `:389` and the emit at
`:405-406`:

```cpp
const double X = Col * ColStep + RowOffsetX + (UnitJitterX * 2.0 - 1.0) * JitterCm;
const double Y = Row * RowStep + (UnitJitterY * 2.0 - 1.0) * JitterCm;
```

X and Y are pure functions of the world axes plus a per-axis box offset. There is no term any
parameter could turn.

**Spacing has no floor, only a jitter ceiling.** `ScatterLayoutMaxJitter = 0.5` at `:57-59`, with the
rationale *"Past half a step a point crosses into its neighbour's cell and the lattice stops
guaranteeing anything about spacing, so the fraction is refused rather than clamped"*, refused at
`:263-270`. Jitter is applied as a per-axis box — `JitterCm = Jitter * Spacing` at `:377`, applied at
`:405-406` — and the emit loop `:385-440` contains **no post-emission distance test of any kind**.
Inside one lattice the cap is sufficient. Across merged lattices there is nothing.

## Measured, and these numbers are the whole case

Nearest-neighbour **bearing** chi-square, normalised to N = 1000, critical value 24.7:

| build | chi-square |
|---|---|
| one lattice at the `0.18` default | **888** |
| 2 lattices merged | 87 |
| 3 lattices merged | 53 |
| 13 lattices + density mask + radius cull | **7.0** |

So the merge workaround does eventually reach an isotropic bearing distribution — at 13 lattices, a
density mask and a radius cull, which is a scatter pipeline rather than a call.

**And it destroys the property the verb does guarantee.** On a 700 cm spacing, merged lattices give
`nnMin` of **10-24 cm**, and **50.2%** of points fall inside half the effective spacing — worse than
`jitter: 0.5`, the value the verb refuses to exceed, which gives 38.9%. The merge is not a cheaper
route to the same output; it is a different, worse output.

**Merging also cannot decorrelate orientation**, because X and Y at `:405-406` are pure functions of
world axes: every merged lattice contributes the same preferred bearings, so the residual is only
diluted, never removed. Rotating the lattices offline — the operation the verb cannot do — took a
4-lattice merge from 54 to **38.6** on the same measure. That single number is what the
`latticeRotation` ask rests on.

**Therefore: the merge workaround is not a substitute for the feature.** It trades a bearing artefact
for a spacing artefact that is measurably worse than the one the verb already refuses to produce.

## Ask, in the order most likely to be accepted

### 1. `latticeRotation` — a rotation of the lattice about the world origin

This is the ask the 54 -> 38.6 measurement directly supports, and **nothing in the project's scatter
doctrine touches orientation**, so it is unopposed. It also does not disturb the two properties the
handler's own header (`:25-34`) calls load-bearing: reproducibility is untouched, and the rotation is
a global constant like the spacing.

**Name the real constraint honestly rather than letting a fixer discover it mid-implementation.** A
rotation applied at emit breaks the "two overlapping regions agree exactly on their intersection"
property (`:30-34`), because the region-bounds solve at `:325-336` walks a world-axis-aligned index
range: two regions rotated the same way still enumerate different `(Col, Row)` sets for their shared
area unless the rotation is part of what identifies a point. **The fix is to fold the rotation into
the per-point key** at `:147-153` (`ScatterLayoutPointStream(Seed, Column, Row)`), and to solve the
index range against the *inverse-rotated* region bounds so the same lattice site is enumerated by
both regions. That is a real change to two functions, not a multiply at the end of the emit loop, and
a fixer who starts by adding the multiply will get a plausible result that silently loses the
intersection guarantee.

### 2. Report spacing statistics — do not add a minimum-distance sampler

Echo `nnMin`, `nnMean`, and the fraction of points inside half the spacing in the `layout{}` block,
beside `rowStep` (`:458`). **A reported floor changes no distribution**, so it cannot be declined on
the doctrine line below, and it is what makes the merge workaround's failure visible to the caller
who is using it — today nothing in the response would have told them about the 10 cm `nnMin`.

### 3. Optionally, a validator form of the floor

Given previously-emitted point sets, report or refuse points within `minSpacing` of one. That is a
**check, not a generator**, so the doctrine's argument does not reach it: it never chooses where a
point goes.

## Why this and the doctrine are not in conflict

`Docs/wiki-src/level-building.instancing-and-scatter.md:74` argues against the obvious ask:

> **Dart-throwing with a minimum-distance test is the wrong tool.** Rejection sampling averages far
> above its own minimum spacing, so the stand comes out visibly sparse and clumped. A **jittered hex
> grid** gives direct control …

And that argument is *enforced by a green test*:
`PinWright.spatial.scatter_layout.JitterIsContinuousAndBounded`
(`Source/PinWright/Private/Tests/Spatial/TestScatterLayout.cpp:237-239`) asserts at `:347-349` that
mean nearest-neighbour distance stays in `(0.6, 1.05) x spacing`, under the comment at `:327-329`:
*"Rejection sampling with a minimum-distance test - the tool the doc page argues against - averages
far above its own minimum, which is why the stand it produces reads sparse and clumped."*

**The doc's argument is about the mean; this ticket is about the minimum and about orientation —
opposite tails of the same distribution.** Rejection sampling is rejected because it pushes the
*mean* nearest-neighbour distance far above the spacing, producing a sparse, clumped stand. Nothing
here proposes moving the mean. `latticeRotation` is a rigid rotation: it preserves every pairwise
distance exactly, so `MeanNearest` is invariant and the assertion at `:349` passes unchanged. Reported
statistics change no point at all. There is no version of either ask that a rejection sampler would
satisfy and no version that this test would newly fail.

**One honest note on the file's stated purpose:** the test's *header* comment (`:10-15`) frames the
whole file around a different defect — a discrete `(-1, 0, 1)` jitter — and the min-distance guard is
one assertion inside it (`:327-349`), not the test's headline. Both readings support the argument
above; the citation is to the assertion, not to the file.

## Cross-links

- `F-scatter-layout-verb` (**DONE**) — built the verb this ticket extends. It closed with these
  distribution measurements recorded; this is the gap they exposed, not a reopening.
- `F-ism-per-instance-transforms` (IN-REVIEW, High) — the consumer side: a layout's transforms are
  only useful if the instances they produce can be addressed afterwards.
- `B-ground-instances-default-component-foreign-scatter` (OPEN, Critical) — the verb callers reach
  for immediately after this one, and the reason a scatter pipeline built on merged lattices is
  currently expensive to correct in place.

## Severity

**Medium.** Impact class: *soft blocker — doable, but only via a documented workaround, a source
dive, or many extra calls*. An isotropic scatter is reachable, at 13 lattices with a density mask and
a radius cull, and the caller has to discover that shape themselves because nothing in the verb or
its docs describes it.

**High declined.** High's neighbouring clause is *hard blocker with no workaround, so a reasonable
task is impossible*. Orientation alone would qualify — it is expressible on no axis the verb has —
but the task the caller actually has ("a stand that does not read as a grid") is achievable by
merging, at 7.0 against a critical 24.7. It is expensive and it silently costs the spacing floor,
which is Medium's description exactly, not High's.

**Low declined.** This is not friction: no amount of reading docs or re-issuing the call produces a
rotated lattice, and the failure the workaround introduces (50.2% of points inside half spacing)
is invisible in the response, so a caller cannot even see the price they paid.

**Reach modifier declined, both directions.** `spatial.scatter_layout` is the only layout verb and is
reached for once per scatter — common in vegetation and set-dressing work, absent from every other
kind of session, so no upward bump. Not a rare edge path either: the artefact is present at the
verb's *default* jitter (888 against a critical 24.7 on a single lattice), so every caller who does
not merge is shipping it.

## History
- `#1-no-lattice-rotation-no-spacing-floor` `OPEN` reporter — Complete param list re-derived at
  `ScatterLayoutHandler.cpp:172-216`: no rotation, no `minDistance`, and the dispatcher rejects
  undeclared keys. Orientation is welded to world axes by `:325-336` and `:405-406`; the jitter cap
  (`:57-59`, refused `:263-270`) bounds spacing within one lattice and the emit loop `:385-440` has no
  distance test across merged ones. Measured (NN-bearing chi-square, N=1000, critical 24.7): one
  lattice at the 0.18 default 888; 2 merged 87; 3 merged 53; 13 lattices + density mask + radius cull
  7.0 — but merged lattices give `nnMin` 10-24 cm on 700 cm spacing and 50.2% of points inside half
  the effective spacing, worse than the 38.9% at `jitter:0.5` the verb refuses to exceed. Merging
  cannot decorrelate orientation (X/Y are pure functions of world axes); rotating lattices offline
  took a 4-lattice merge from 54 to 38.6. Asks: `latticeRotation`, with the honest constraint that it
  must be folded into the per-point key at `:147-153` and the bounds solve at `:325-336` or the
  overlapping-regions guarantee at `:30-34` is silently lost; reported `nnMin`/`nnMean`/half-spacing
  fraction in `layout{}` beside `rowStep` (`:458`); optionally a validator form of the floor.
  Deliberately **not** asking for a minimum-distance sampler — checked against
  `Docs/wiki-src/level-building.instancing-and-scatter.md:74` and against the green assertion at
  `TestScatterLayout.cpp:347-349` (comment `:327-329`), both of which argue about the *mean* while
  this argues about the *minimum* and about orientation. A rigid rotation preserves every pairwise
  distance, so that assertion passes unchanged.
