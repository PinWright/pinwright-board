---
id: E-ground-instances-embed-fraction-of-bounds-height
title: "`embedFraction`'s height term is the ROTATION-INFLATED world AABB height, not the mesh height times scale — measured 2898.7 uu against 2805.1 for a tree at 1.9 degrees of pitch — so the same instance beds deeper as it tilts; and because the default 0.02 is a fraction of height, one parameter value sinks a tree 58 cm and a groundcover plant 1 cm, while the batch `seat` echo reports the inputs and never the resolved centimetres"
status: OPEN
severity: Low
category: ergonomic
tags: [spatial, ground_instances, ground_actors, embed-fraction, embed-depth, bounds, aabb, rotation-inflated, docs, seat-echo, batch-echo, defaults, placement, vegetation]
encounters: 1
lastSeen: 2026-08-29T20:10:00+03:00
---

# "Bounds height" is doing more work than the phrase admits

`embedFraction` is documented, defaults non-zero on purpose, and its resolved value reaches the
caller. What is not stated anywhere is *which* height it is a fraction of. It is the **world-space
axis-aligned bounding box** height, re-fitted after the object's rotation — so the same object at the
same scale beds deeper the more it is tilted, and a hand-computed check against `meshHeight * scaleZ`
disagrees with the verb.

The second half is a defaults question rather than a documentation one: a fraction of *height* is a
strange scale for a quantity whose stated purpose is contact. The parameter's own justification
(`GroundPlacementHandler.cpp:804-806`, *"an object resting exactly tangent to terrain reads as
balanced rather than placed"*) is about the few centimetres where an object meets the ground, and a
19 m tree and a 0.5 m plant need the same few centimetres.

## Mechanism, re-derived at HEAD `6d0e91a3`

All paths in `Plugins/PinWright/Source/PinWright/Private/Handlers/Spatial/`.

    GroundPlacementUtils.cpp:443-447   double FGroundSeatConfig::ResolveEmbedCm(double BoundsHeightCm) const
                                       {
                                           const double FromFraction = EmbedFraction * FMath::Max(BoundsHeightCm, 0.0);
                                           return FMath::Max(EmbedDepthCm + FromFraction, 0.0);
                                       }

The height term at each call site:

    GroundPlacementUtils.cpp:1509   (instances) Config.ResolveEmbedCm(2.0 * InstanceBounds.GetExtent().Z);
    GroundPlacementUtils.cpp:1316   (actors)    Config.ResolveEmbedCm(2.0 * Extent.Z);

`InstanceBounds` is `Mesh->GetBounds().GetBox().TransformBy(InstanceWorld)` (`:1450`), and
`FBox::TransformBy` returns the axis-aligned hull of the **rotated** box. `Extent` on the actor path
comes from `Actor->GetActorBounds(false, Origin, Extent)` (`:1195`), which is likewise a world AABB.
So on both verbs the height grows with pitch and roll.

Defaults and reads: `DefaultEmbedFraction = 0.02` (`GroundPlacementUtils.h:91`, field `:538`), read at
`GroundPlacementHandler.cpp:1432-1434` (instances) and `:899-901` (actors), both `FMath::Max(..., 0.0)`
with **no upper clamp**. The absolute sibling `embedDepth` defaults `0`
(ParamSpec `:1299`, `:808`).

Both ParamSpecs say "bounds HEIGHT" in capitals and neither says which bounds:

    GroundPlacementHandler.cpp:1295-1298   "Sink this fraction of the instance's bounds HEIGHT into the
                                            ground after the seat solve, so it beds in rather than
                                            resting tangent. Set 0 for a pure tangent rest."
    GroundPlacementHandler.cpp:803-807     "Sink this fraction of the actor's bounds HEIGHT into the
                                            ground after the seat solve. Non-zero by default on purpose:
                                            an object resting exactly tangent to terrain reads as
                                            balanced rather than placed. Set 0 for a pure tangent rest."

## Measured

Host `EAContentExamples58`, UE 5.8, `/Game/Maps/PW_VegetationTest`;
`Docs/map/tree-seating-on-slopes.md`, producers under `dev/planting/`.

- A `HillTree_P2` instance at scale 1.48 with **1.9 degrees** of pitch measures a world AABB height of
  **2898.7 uu** against **2805.1** for `meshHeight * scale`. 3.3% from two degrees of tilt, on a mesh
  whose horizontal half-extents (656.8 x 801.3) are large relative to its height — the inflation is
  proportional to the horizontal extent, so it is worst exactly for the wide-crowned meshes.
- At the default `embedFraction: 0.02` that resolves to **58 cm** of sink on this tree and **1 cm** on
  a 50 cm `S_Nordic_Coastal_Groundcover` plant, from the same parameter value in the same call.
- The 58 cm is not harmless bookkeeping on this project: it **partially masks** the seat error that
  `B-ground-instances-footprint-is-bounds-not-contact` describes, which is why that defect reads as
  intermittent — a tall tree is being sunk half a metre by a default the caller never set, and on
  shallow slopes that cancels the lift.

## What is and is not already reported — premise corrected during filing

The report this ticket was filed from implied the resolved embed is invisible. **It is not, and the
correction shrinks this ticket.** `Result.Seat.AppliedEmbedCm` is serialized per row as `embedCm` at
`GroundPlacementHandler.cpp:1523` and `:1535` (instances) and `:565` (actors), set at
`GroundPlacementUtils.cpp:1522` / `:1554` / `:1332`. A caller reading rows can see the centimetres.

What is genuinely missing:

1. **Neither ParamSpec, nor `docs/wiki-src/spatial.md`, nor `docs/wiki-src/spatial.ground-placement.md`
   says the height is the rotation-inflated world AABB height.** A caller predicting the seat by hand
   — which is what a dry run invites — uses `meshHeight * scaleZ` and is wrong by a tilt-dependent
   amount they cannot see in the inputs.
2. **The batch `seat` echo reports the inputs and not the output.**
   `GroundPlacementHandler.cpp:1563-1574` (instances) and `:981-994` (actors) echo `embedFraction` and
   `embedDepthCm`; the resolved centimetres appear only in `results[]`, which `detail` governs and
   which is capped at 256 rows. At `detail: "summary"` — the setting a caller uses on a batch of 840 —
   the resolved embed is not in the response at all, so a batch-wide policy is only checkable row by
   row. An `embedCmRange` or `embedCmMedian` in the `seat` echo closes that.
3. **The default's scale is arguable and the ticket says so rather than asserting a fix.** A fraction
   of height means "sink 2% of how tall you are", when the stated intent is "do not rest exactly
   tangent". `embedDepth` already exists as the absolute form. Whether the default should move to a
   small absolute value (this project's offline seat uses `0.10 * contactRadius`, an absolute few
   centimetres proportional to the *contact* patch, and measured 0/20 floating against 20/20 before)
   is a design call for whoever owns these defaults — but the current default is the one that makes a
   19 m tree and a 0.5 m plant behave differently for no reason a caller stated.

## Ask

- One sentence in both ParamSpecs and in the `spatial.ground-placement.md` embed prose: the height is
  the object's **world bounding-box** height, re-fitted after rotation, so a tilted instance embeds
  deeper than `meshHeight * scale` predicts.
- An aggregate resolved-embed figure in the batch `seat` echo, so `detail: "summary"` is not blind to
  it.
- Consider an absolute default (`embedDepth`-based) with `embedFraction` defaulting 0. Flagged for
  decision, not asserted as the fix.

## Scope — deliberately separate from the footprint ticket

This is **not** a sub-item of `B-ground-instances-footprint-is-bounds-not-contact`, even though both
are "the AABB is the wrong shape to measure from" and both were found in the same pass. The deciding
reason is that this one applies identically to `spatial.ground_actors` (`GroundPlacementUtils.cpp:1316`,
`:1195`), which that ticket does not touch and which has a working `undersideModel: "mesh"` escape
from the footprint problem and no escape from this one. Filing it inside a `ground_instances`-only
ticket would hide a `ground_actors` defect. They do interact — the 58 cm masks that ticket's lift —
and that interaction is recorded in both.

Related: `E-bounds-plane-fallback-has-no-batch-level-count` makes the same complaint one field over —
a batch echo that reports what was *requested* rather than what was *resolved*. Whoever fixes the
`seat` echo there should do this one in the same edit.

## Severity

**Low.** Impact class is the rubric's Low band verbatim — *docs, discoverability, naming*. The
parameter does what its name says, its resolved value is reported per row (`embedCm`,
`GroundPlacementHandler.cpp:565`/`:1523`/`:1535`), and nothing here is a wrong result: a caller who
reads the rows can see exactly how deep everything went.

**Not Medium**, and this is a correction to how the finding was reported to the board rather than a
judgement call. Medium would need *a readback that omits a field and forces a fallback*; the readback
does not omit it, only the batch summary does, and `detail: "failures"` (the default) already carries
rows. The remaining gap is that the *derivation* is undocumented, which is friction.

**Reach modifier declined.** `embedFraction` has a non-zero default, so it is applied on every seat
call on both grounding verbs whether the caller passes it or not — a real argument for the bump. It is
declined because the rubric's bump is for methods that run in almost every session, and the grounding
verbs are a level-building path rather than a universal one; a default that is silently applied is
still only applied when the verb is called. Low stands.

## History
- `#1-embed-height-is-the-rotated-aabb` `OPEN` reporter — Filed from a planting-diagnosis pass on
  `/Game/Maps/PW_VegetationTest` (host `EAContentExamples58`, UE 5.8), method in
  `Docs/map/tree-seating-on-slopes.md`. Mechanism re-derived at HEAD `6d0e91a3`:
  `FGroundSeatConfig::ResolveEmbedCm` (`GroundPlacementUtils.cpp:443-447`) multiplies `EmbedFraction`
  by a `BoundsHeightCm` supplied as `2.0 * InstanceBounds.GetExtent().Z` (`:1509`) on the instance
  path and `2.0 * Extent.Z` (`:1316`, from `Actor->GetActorBounds` at `:1195`) on the actor path —
  both world AABBs, and the instance one is `TransformBy(InstanceWorld)` (`:1450`), i.e. the
  axis-aligned hull of the rotated box. Default `0.02` (`GroundPlacementUtils.h:91`), read with no
  upper clamp at `GroundPlacementHandler.cpp:1432-1434` / `:899-901`. Both ParamSpecs (`:1295-1298`,
  `:803-807`) say "bounds HEIGHT" and neither says which bounds; the wiki pages say nothing about the
  derivation either. Measured: a scale-1.48 tree at 1.9 degrees of pitch has a world AABB height of
  **2898.7 uu** against **2805.1** for `meshHeight * scale`, so the embed grows with tilt; the same
  default value sinks that tree **58 cm** and a 50 cm groundcover plant **1 cm**, and the 58 cm
  partially masks the lift described in `B-ground-instances-footprint-is-bounds-not-contact`, which is
  why that defect reads as intermittent. **Premise corrected during filing:** the report implied the
  resolved embed is invisible to the caller. It is not — `Result.Seat.AppliedEmbedCm` is serialized as
  `embedCm` per row (`GroundPlacementHandler.cpp:565`, `:1523`, `:1535`; set at
  `GroundPlacementUtils.cpp:1332`, `:1522`, `:1554`). That correction is why this is rated Low rather
  than Medium: the surviving gaps are the undocumented derivation, the batch `seat` echo reporting
  inputs (`embedFraction`, `embedDepthCm`) without any resolved figure so `detail: "summary"` is blind,
  and the open design question of whether the default should be absolute. Filed separately from the
  footprint ticket on the deciding ground that it applies identically to `spatial.ground_actors`,
  which that ticket does not cover. Reach bump declined with the argument stated.
