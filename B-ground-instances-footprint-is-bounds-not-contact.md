---
id: B-ground-instances-footprint-is-bounds-not-contact
title: "`spatial.ground_instances` samples its seat grid over the instance's world AABB and models the underside as a flat plane at that box's minimum, so for anything whose contact patch is not its silhouette it seats the BOX and not the mesh: 177/177 correctly-planted trees are proposed for a lift, p50 +196 cm and max +773 cm, while every row reports `coverage 1.00`, `pass true` and `undersideReliefCm 0` — and no combination of `footprintInset` / `seatPercentile` / `embedFraction` can reach the trunk"
status: OPEN
severity: High
category: bug
tags: [spatial, ground_instances, ism, hism, instanced-static-mesh, scatter, vegetation, trees, bounds, aabb, footprint, contact-patch, underside, bounds-plane, silent-wrong-data, false-pass, placement, level-building, no-workaround]
encounters: 2
lastSeen: 2026-08-29T20:50:00+03:00
---

# The footprint is the silhouette, and for a tree the silhouette is the canopy

`spatial.ground_instances` decides *where to probe* and *what the bottom of the object looks like*
from one source: the instance's world-space axis-aligned bounding box. That is a fair model for a
rock, a slab, a log or a boulder. It is wrong by **46.7x in area** for `HillTree_P2` and **80.8x**
for `SM_Dead_Tree`, because a trunked tree's bounding box is its canopy and its contact geometry is
a root flare two centimetres wide by comparison.

The verb does not report this, and cannot be argued out of it: the parameter that looks like the
remedy is clamped well short of what would be needed, and the fields a caller would gate on
(`coverage`, `pass`, `undersideReliefCm`) are all truthful **about the box** and therefore all
green.

## Mechanism, re-derived at HEAD `6d0e91a3`

All paths in `Plugins/PinWright/Source/PinWright/Private/Handlers/Spatial/`.

**The footprint is the AABB.** `GroundPlacement::SeatInstance` builds the box and hands it to the
sampler:

    GroundPlacementUtils.cpp:1450   const FBox InstanceBounds = Mesh->GetBounds().GetBox().TransformBy(InstanceWorld);
    GroundPlacementUtils.cpp:1470   FGroundContactReport Pre = MeasureContactForBounds(World, InstanceBounds, nullptr, Ignore,

`FBox::TransformBy` returns the axis-aligned hull of the *rotated* box, so instance yaw inflates it
further. `MeasureContactForBounds` then derives the whole sample grid from that box and nothing
else:

    GroundPlacementUtils.cpp:922-923   const FVector Origin = WorldBounds.GetCenter();
                                       const FVector Extent = WorldBounds.GetExtent();
    GroundPlacementUtils.cpp:933       const double Inset = FMath::Clamp(FootprintInset, 0.0, 0.45);
    GroundPlacementUtils.cpp:934-937   const double HalfX = Extent.X * (1.0 - Inset);
                                       const double HalfY = Extent.Y * (1.0 - Inset);
                                       const double TopZ = Origin.Z + Extent.Z;
                                       const double BottomZ = Origin.Z - Extent.Z;

No contact geometry, no underside profile and no caller-supplied extent participates in choosing
sample XY.

**The underside is a flat plane at the box minimum.** The instance path passes
`EUndersideModel::BoundsPlane` as a literal at both measurement sites — the pre-measure
(`GroundPlacementUtils.cpp:1471`) and the post-move readback (`:1564`) — and the handler assigns it
unconditionally with no parameter read (`GroundPlacementHandler.cpp:1428`,
`Config.UndersideModel = GroundPlacement::EUndersideModel::BoundsPlane;`). In that branch every
column's underside is the same number:

    GroundPlacementUtils.cpp:996-1000   else
                                        {
                                            Column.bHasActorGeometry = true;
                                            Column.UndersideZ = BottomZ;
                                        }

`BottomZ` is `:937`, i.e. the AABB minimum. So for a tree the solve believes the object's bottom
face is a 13.1 m x 16.0 m flat rectangle sitting at the lowest point of the canopy box, and it seats
*that*.

**The hardcode itself is correct and this ticket does not reopen it.**
`GroundPlacementHandler.cpp:1416-1418` states the reason (an instance has no per-column probeable
underside, because `UInstancedStaticMeshComponent::LineTraceComponent` answers from every body at
once), it is echoed honestly at `:1573`, and `F-ground-instances-align-to-surface` already declined
to question it. What is wrong is not the *underside model* — it is that the **footprint** the model
is applied over is the silhouette, with no way to say otherwise.

## Measured: the mesh geometry, from its own LOD0 vertices

`Docs/map/tree-seating-on-slopes.md` (host project `EAContentExamples58`, UE 5.8), producer
`dev/planting/p_meshprofile.py`, output `dev/planting/out/meshprofile.json`. Contact half-extent is
measured over the lowest 6% of mesh height.

| mesh | bounds half-extent XY | contact half-extent XY | bounds/contact **area** |
|---|---|---|---|
| `HillTree_P2` | 656.8 x 801.3 | 109.7 x 102.8 | **46.7x** |
| `SM_Dead_Tree` | 42.5 x 42.0 | 4.7 x 4.7 | **80.8x** |
| `S_Forest_Rock_Shelf...Var1` | 278.3 x 111.5 | 278.4 x 114.2 | 0.98x |
| `SM_Driftwood` | 237.0 x 49.5 | 231.4 x 44.3 | 1.14x |
| `SM_MT_Boul_A` | 4165.0 x 3586.2 | 4232.1 x 2731.7 | 1.29x |
| `SM_Rock` | 21.3 x 21.3 | 11.9 x 15.3 | 2.49x |

**That table is the scope of the defect stated precisely.** The AABB footprint is a good model for
four of these six meshes and a 47-81x overestimate for the two with a trunk. This is not "the verb is
wrong"; it is "the verb has one footprint source and no way to name the other one".

`HillTree_P2` detail: mesh bottom is z -37.5; the root flare spans x[-94.8, 109.7] y[-102.8, 94.3],
centred on the pivot; the trunk above it is r ~= 60; the **first canopy geometry appears at z +390**
and the canopy is full width by z +485. The bounds origin is `(-4.8, -167.8, 912.1)` — the box
centre is 167.8 uu off the trunk axis in Y before any yaw is applied.

## Measured: what the verb proposes, from its own dry runs

`spatial.ground_instances {actorName:"Actor_3", component:"HISM_ZF_HillTree",
surface:{preset:"landscape"}, apply:false}` over 177 instances that are **already correctly seated**
(field measurement below: this component's gap p50 is -15.1 cm, i.e. bedded).

| parameters | proposed dZ p50 | p90 | max | notes |
|---|---:|---:|---:|---|
| defaults (`samples:3, footprintInset:0.1, seatPercentile:0, embedFraction:0.02`) | **+196.0** | +564.6 | +773.4 | **lifts 177/177**; 56 by over 3 m |
| `samples:1` | +41.2 | +89.3 | +157.4 | see `B-ground-instances-single-sample-degenerate` — every quality field is vacuous at this setting |
| best reachable: `samples:3, footprintInset:0.45, seatPercentile:0.5, embedFraction:0` | +76.9 | +149.3 | +250.0 | only **9/177** within 20 cm of a no-op |

**Every one of those runs reports `coverage: 1.00`, `pass: true`, `undersideReliefCm: 0`.** Expected
proposed dZ is approximately 0 — these instances were already on the ground.

### `footprintInset` is not the missing parameter, and the proof matters

The obvious review response is "use `footprintInset`". It cannot reach:

- The clamp is `0.45`, enforced three times — `GroundPlacementUtils.cpp:933` (the one that governs
  sampling), plus read-side clamps at `GroundPlacementHandler.cpp:1424-1427` (instances) and `:623-626`
  (actors). Default `0.10` (`GroundPlacementUtils.h:86`, ParamSpec `GroundPlacementHandler.cpp:1286-1289`).
- At the ceiling, `HalfX`/`HalfY` (`:934-935`) become `656.8 x 0.55 = 361.2` and `801.3 x 0.55 = 440.7`
  uu — still **361 x 441 uu of sample half-extent around a 100 uu contact radius**, i.e. still 16x
  the contact area.
- Measured at that ceiling (row 3 above): p50 **+76.9 cm**, max **+250.0**. Better than the default
  and still a lift on 168 of 177.

`footprintInset` is a fraction of the wrong quantity. To express a 100 uu contact radius on a 656.8 uu
half-extent it would need to reach 0.848, and even then it would still be a *ratio of the bounds*,
which changes per instance with yaw and scale rather than staying fixed to the mesh.

## Measured: the field consequence, 840 instances

`dev/planting/p_gap.py` probes the landscape (landscape-only ignore list, complex trace) at the pivot
and on a ring of radius `100 x scale` at 8 azimuths; `gapWorst = treeBottomZ - min(ringGroundZ)`,
positive meaning daylight under the trunk.

Level-wide for `HillTree_P2`: **254/840 (30.2%) have daylight under the trunk, 93 (11.1%) over 30 cm,
22 (2.6%) over 1 m, worst 252 cm.** It correlates with slope exactly as the mechanism predicts —
0-5 deg: 24% over 10 cm; 25-30 deg: 62%; 35-44 deg: 85%, p50 +76.5 cm.

The same effect on the other classes, all seated through this verb:

| class | floating | gap p50 | gap max |
|---|---|---:|---:|
| rock shelves | 224/238 (**94%**) | +41.4 | +331.6 |
| driftwood | 22/27 (**81%**) | +49.2 | +254.8 |
| `SM_Rock` | 355/466 (**76%**) | +12.6 | +56.7 |

Those three have bounds/contact ratios of 0.98-2.49x, so their residual is the *defaults* problem
(`seatPercentile: 0` is a first-contact rest), not this one — recorded here only to show the
measurement is not tree-specific instrumentation. **The 47-81x class is what this ticket is about.**

### The fix direction is measured, not proposed

A 20-instance pilot on the steepest cluster (slope p50 25.1 deg) computed the seat offline against
the *contact* radius and wrote it with `actor.set_instance_transforms`:

    r     = contactR * meanXYscale          # HillTree_P2 contactR = 100 uu
    newZ  = min(ground at 16 azimuths on r) - bottomOff * scaleZ - embed
    embed = 0.10 * r                        # ABSOLUTE, proportional to contact radius

`gapWorst` went from +41.8/+105.0/+233.9 (min/mean/max, 20/20 with daylight) to
-17.6/-9.7/-4.5 (**0/20 with daylight**), and prediction matched measurement within 5.6 cm worst
case. Vision-verified at three low downhill poses (1280x720, `ev100 -0.5`): before, each trunk ends
in mid-air at a flat dark horizontal cut with lit grass visible underneath it; after, the trunk runs
continuously into the grass. So the solve this verb already performs is correct — only the extent it
performs it over is wrong.

## Ask

Either shape; the first is smaller and the second is more general.

1. **`contactRadius`** (number, mesh-local cm, scaled per instance by the instance's mean XY scale).
   When set, `MeasureContactForBounds` derives `HalfX`/`HalfY` from it instead of from
   `Extent.X/Y * (1 - Inset)` at `GroundPlacementUtils.cpp:934-935`, leaving `TopZ`/`BottomZ` and the
   whole rest of the solve untouched. Mesh-local rather than a bounds ratio, so yaw and per-instance
   scale do not move it.
2. **`footprintSource: bounds | contact | custom` + `contactExtent`**, where `contact` derives the
   XY half-extent from the mesh's own lowest-slice vertices (what `dev/planting/p_meshprofile.py`
   computes offline) and `custom` takes an explicit `contactExtent`. This also gives the response
   somewhere honest to report which source was used, the way `undersideModel` is already echoed at
   `GroundPlacementHandler.cpp:1573`.

Whichever lands, the **response must name the footprint it sampled** — a `footprintHalfExtentCm` pair
beside the existing `seat` echo. The whole failure here is that three green fields describe a box the
caller never asked about, and none of them names it.

`spatial.ground_actors` needs the same parameter for the same reason (its footprint comes from
`Actor->GetActorBounds`, `GroundPlacementUtils.cpp:1195`), but it has `undersideModel: "mesh"` as a
partial escape and instances do not, so the instance verb is where this is a blocker.

## Not a duplicate of

- **`F-ground-instances-align-to-surface`** (OPEN, Medium) — that ticket is about the *rotation* the
  solve computes and discards; this is about the *XY extent* the solve samples over. A fixer could
  land either without the other. Its § *Scope* paragraph declines to reopen the `bounds_plane`
  underside hardcode, and so does this ticket, for the same reason.
- **`E-grounding-coverage-cannot-catch-a-single-point-rest`** (OPEN, Medium) — `coverage` being a
  ratio over a collapsible denominator. Here `coverage` is 1.00 with `actorColumns: 9`, all nine
  columns genuinely supported: the coverage number is *right*, and the seat is still 196 cm out. That
  is why `minSupportedColumns` — that ticket's ask — would not catch this one either.
- **`E-bounds-plane-fallback-has-no-batch-level-count`** (OPEN, Medium) — the mesh-profile probe
  *falling back* to a bounds plane without a batch count, and `undersideReliefCm` reading 0 on those
  rows. On `ground_instances` there is no fallback: `bounds_plane` is the only model there has ever
  been (`GroundPlacementHandler.cpp:1428`). The zeroed `undersideReliefCm` this ticket reports is the
  same *symptom* arriving down a different path, so a fix to that ticket's batch echo does not touch
  this.
- **`B-verify-grounding-maxgap-false-fail`** (DONE, High) — decomposed clearance into underside
  relief plus float so overhanging geometry stops failing a well-seated actor. That fix works from
  the underside samples; here the underside samples are all one number by construction, so the
  decomposition has nothing to decompose.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as* — `B-ground-probe-hits-hull-not-render`,
`B-niagara-validate-green-while-component-inactive`, `B-mrq-render-result-omits-bitrate-and-size`:
the call succeeds, every number it reports is correct, and the output is wrong because the deciding
number was never reported. This is the strongest instance of it yet found on the spatial verbs,
because the unreported number is not merely absent — it is *not expressible*: no parameter of the
verb can name the contact footprint, so a caller cannot even ask the question the answer depends on.

Nearest sibling in mechanism is `B-ground-probe-hits-hull-not-render` (DONE, High): there the probe
resolved against a collision hull that was not the render mesh; here the probe resolves against a
bounding box that is not the contact patch. Both answer a geometrically valid question about the
wrong solid.

## Severity

**High.** Two of the rubric's bands are hit and they agree:

- *Silent wrong data on a normal path.* `pass: true`, `coverage: 1.00`, `undersideReliefCm: 0` on
  every one of 177 rows that propose lifting a correctly-planted tree by a median of 196 cm. The
  caller trusts the result and builds on it — measurably: 30.2% of this level's `HillTree_P2` have
  daylight under the trunk, and the review that missed it read exactly those three fields.
- *Hard blocker with no workaround.* Established by exhaustion rather than asserted: the three
  parameters that touch the footprint were each driven to their limits and measured
  (`footprintInset` to its 0.45 clamp, `seatPercentile` to 0.5, `embedFraction` to 0), the best
  reachable result is still +76.9 cm p50 with 9/177 acceptable, and `samples: 1` — the workaround this
  project actually adopted — trades the problem for a different one
  (`B-ground-instances-single-sample-degenerate`). The only working route is to leave the verb
  entirely and compute the seat offline against `actor.set_instance_transforms`, which is
  reimplementing the verb.

**Not Critical.** Nothing is corrupted and nothing is unrecoverable: `apply:false` writes nothing, and
an applying seat returns every original transform in `movedInstances[]`. Worth noting against that:
`B-foliage-mutators-no-transaction` means the receipt is the *only* record, so a caller who discards
the response has no undo — that raises the cost of the mistake but does not make the write itself
destructive.

**Reach modifier declined, in the direction that would raise it, and the argument stated.**
`spatial.ground_instances` is the only per-instance grounding verb and `HOLDER_NOT_SEATABLE`
(`GroundPlacementHandler.cpp:718` on `ground_actors`, `:1038` on `verify_grounding`) routes every
ISM/HISM scatter caller here by construction — and instanced scatter is overwhelmingly *vegetation*,
which is the affected class. That is a real case for the bump. It is declined because the measured
mesh table above shows the verb is correct for four of six sampled classes (0.98-2.49x), so this is a
defect on a large subset of the verb's traffic rather than on all of it, and High is already the
right band without it.

## History
- `#1-footprint-is-the-silhouette` `OPEN` reporter — Filed from a planting-diagnosis pass on
  `/Game/Maps/PW_VegetationTest` (host `EAContentExamples58`, UE 5.8); full method and numbers in
  `Docs/map/tree-seating-on-slopes.md`, producers under `dev/planting/`. Mechanism re-derived at HEAD
  `6d0e91a3` rather than relayed: footprint is `Mesh->GetBounds().GetBox().TransformBy(InstanceWorld)`
  (`GroundPlacementUtils.cpp:1450`) passed to `MeasureContactForBounds` (`:1470`), whose grid comes
  entirely from `WorldBounds.GetCenter()`/`GetExtent()` (`:922-923`) scaled by the inset (`:933-937`);
  underside is the flat AABB minimum in the `BoundsPlane` branch (`:996-1000`, `BottomZ` from `:937`),
  and `BoundsPlane` is a literal at both measurement sites (`:1471`, `:1564`) with the handler
  assigning it unconditionally at `GroundPlacementHandler.cpp:1428`. **Two relayed citations were
  wrong and are corrected here:** `:1450` computes the AABB, it does not compute the underside plane
  (that is `:999`); and the `BoundsPlane` literals are at `:1471`/`:1564`, not `:1470`/`:1563`.
  Measured mesh geometry from LOD0 vertices gives bounds/contact **area** ratios of 46.7x
  (`HillTree_P2`) and 80.8x (`SM_Dead_Tree`) against 0.98-2.49x for rocks, slabs, logs and boulders —
  which is the exact scope of the defect. Dry runs over 177 already-seated instances propose a lift on
  **177/177**, p50 +196.0 / p90 +564.6 / max +773.4 cm, while reporting `coverage 1.00`, `pass true`,
  `undersideReliefCm 0`. `footprintInset` proven not to be the remedy rather than assumed: clamped
  0.45 at `GroundPlacementUtils.cpp:933` (read-side `GroundPlacementHandler.cpp:1424-1427`), which
  still leaves 361 x 441 uu of sample half-extent around a 100 uu contact radius, and the best
  reachable parameter set measures p50 +76.9 with 9/177 acceptable. Field impact 254/840 (30.2%)
  floating, 93 over 30 cm, worst 252 cm, monotone in slope. Fix direction validated by a 20-instance
  pilot seated offline against the contact radius: 20/20 with daylight became 0/20, prediction within
  5.6 cm, vision-verified at three poses. Ask is `contactRadius` (mesh-local cm) or
  `footprintSource` + `contactExtent`, plus a `footprintHalfExtentCm` echo. Explicitly NOT reopening
  the `bounds_plane` underside hardcode (`GroundPlacementHandler.cpp:1416-1418`), which
  `F-ground-instances-align-to-surface` already examined and accepted. Severity High on two
  concurring bands (silent wrong data on a normal path; hard blocker established by driving every
  footprint parameter to its limit and measuring); Critical declined because `movedInstances[]`
  carries every original transform; reach bump declined with the argument stated, because four of six
  measured mesh classes are served correctly by the bounds footprint.

- `#2-offline-contact-ring-shipped-what-the-verb-could-not-reach` `OPEN` reporter — **Outcome
  evidence, not a new mechanism. Status deliberately unchanged — this is a second encounter, and I
  was not asked to verify anything.** `#1` ended at a 20-instance pilot and a proposed direction;
  the direction has now been carried to every affected instance in the level, and the result is the
  strongest available statement of this ticket's claim: **the capability is achievable, it is just
  not reachable through the verb.** Method and full tables in
  `Docs/map/tree-seating-on-slopes.md` § *Bulk re-seat, applied 2026-08-29* (host
  `EAContentExamples58`, UE 5.8); producers `dev/planting/p_bulk_prep.py`, `p_gap.py`,
  `p_resolve.py`. All figures below re-derived from `dev/planting/out/gap_before.json` and
  `gap_after.json` (the same ring metric on both sides) rather than relayed.
  **Offline contact-ring seat, written with `actor.set_instance_transforms`:**
  `HillTree_P2` 840 instances, floating **234 (27.9%) -> 19 (2.3%)**, gap p90 +27.3 -> **-4.6**,
  max **+252.1 -> +1.9 cm**; `SM_Dead_Tree` 154 instances (defect set only, 27 written), floating
  **27 (17.5%) -> 0 (0.0%)**, p90 +63.6 -> -7.9, max **+136.2 -> -5.4 cm**. 215 of the 840 trees
  were moved; the rest were already bedded and were left alone. The seat is `#1`'s formula
  unchanged: ground minimum on a ring of `contactR * meanXYscale` at 16 azimuths, minus the mesh
  z-min times scale, minus an embed that is **10% of the contact radius** rather than any fraction
  of bounds height.
  **Against that, the verb's own best reachable dry run on the same class is p50 +76.9 cm of
  LIFT** (`#1`, `footprintInset` at its 0.45 clamp, `seatPercentile 0.5`, `embedFraction 0`), with
  9 of 177 instances within 20 cm of a no-op. So the gap between "what this verb can be asked for"
  and "what the same solve produces over the right footprint" is a full 8 metres of tree base, and
  it closes to zero the moment the footprint is nameable. Nothing else changed: the same landscape,
  the same probe, the same complex trace, the same instance rotations and scales.
  **One correction to `#1`'s scope table, which will matter to whoever fixes this.** `#1` says the
  bounds footprint is "a good model for four of these six meshes (0.98-2.49x)". That is true of the
  **XY** footprint and false of the **Z** term of the same box: for the four low-ratio classes the
  seat still went wrong, in the opposite direction, because the underside plane is the
  *rotation-inflated* AABB minimum. Measured on the same bulk pass and filed separately as
  `B-ground-instances-rotated-aabb-underside-plane` — `seatPercentile 0.4` proposed a **lift** on
  245 of 256 already-bedded rocks. A `contactRadius` / `footprintSource` fix as asked for here does
  **not** address that, and neither ticket subsumes the other: this one is the XY extent the solve
  samples over, that one is the Z the solve seats to.
  **Two acceptance rules had to be built offline because the verb offers neither**, which is filed
  as `F-ground-instances-move-clamps`: never lift (105 instances measured worse after the first
  apply, 38 reverted from the pre-recorded originals — count re-derived from the five revert
  payloads in `dev/planting/out/args/`), and never over-bury (11 reverted, one driftwood log driven
  2.7 m further into the terrain with `pass: true` on the row). Neither rule is expressible as a
  parameter, and the applying response publishes no per-instance number at the default `detail`,
  so both had to be enforced by measuring the level again afterwards.
  **Vision-verified, not only measured**, at five fresh poses the pilot never touched (1280x720,
  fov 60, `ev100 -0.5`, `hideEditorSprites: true`). The clearest pair is `HISM_ZF_HillTree` idx 63:
  before, the root flare ends in mid-air with lit grass visible underneath and behind it; after, the
  trunk runs continuously into the terrain. The level's worst tree (`#1`'s +252.1 cm) went from a
  trunk cut off above a black cavity with a rock lit underneath it, to a trunk descending into the
  grass. Honest limits recorded with the evidence: the dead-tree pair's base is below the local
  grass line in **both** frames, so that class's evidence is numeric only; and the rock-shelf pair
  still shows a visible overhang on the downhill lip, which is the honest limit of a flat-plane
  underside on 3 m of relief and is not claimed as a win here.
  `encounters` 1 -> 2. No severity change proposed: same impact class, and a second observation is
  an `encounters` input, never a severity input.
