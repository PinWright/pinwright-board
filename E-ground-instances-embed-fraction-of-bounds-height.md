---
id: E-ground-instances-embed-fraction-of-bounds-height
title: "`embedFraction` ParamSpecs on `spatial.ground_instances` / `spatial.ground_actors` still say only \"bounds HEIGHT\" without naming the rotation-inflated world AABB, and the batch `seat` echo reports the embed inputs but never the resolved centimetres, so `detail: \"summary\"` cannot check it"
status: OPEN
severity: Low
category: ergonomic
tags: [spatial, ground_instances, ground_actors, embed-fraction, embed-depth, bounds, aabb, rotation-inflated, docs, seat-echo, batch-echo, defaults, placement, vegetation]
encounters: 1
costly: 1
lastSeen: 2026-08-29T20:10:00+03:00
rice: [1, 1, 1, 1]
priority: 8
---

# `embedFraction`'s height is the rotated world AABB, which the ParamSpecs do not say, and the batch echo never reports the resolved embed

`embedFraction` (default `0.02`, `Source/PinWright/Private/Handlers/Spatial/GroundPlacementUtils.h:91`)
multiplies the object's **world axis-aligned bounding-box height**, re-fitted after rotation:
`FGroundSeatConfig::ResolveEmbedCm` (`GroundPlacementUtils.cpp:481-485`) is called with
`2.0 * InstanceBounds.GetExtent().Z` on the instance path (`:1720`, box from `TransformBy(InstanceWorld)`
at `:1639`) and `2.0 * Extent.Z` on the actor path (`:1505`, from `GetActorBounds` at `:1384` / `:1461`).
A tilted object therefore beds deeper than `meshHeight * scaleZ` predicts: a scale-1.48 `HillTree_P2`
pitched 1.9 degrees measured 2898.7 uu against 2805.1, and the default resolves to 58 cm on that tree
and 1 cm on a 50 cm groundcover plant (`EAContentExamples58`, UE 5.8, `/Game/Maps/PW_VegetationTest`).

The wiki now explains this for instances (`docs/wiki-src/spatial.ground-placement.md:104`). What is
still missing:

1. **Both ParamSpecs say only "bounds HEIGHT"** — `GroundPlacementHandler.cpp:819-822`
   (`ground_actors`) and `:1365-1368` (`ground_instances`) — and the general embed prose at
   `spatial.ground-placement.md:69` says "the actor's bounds height" with no mention of rotation.
2. **The batch `seat` echo reports inputs, not the result.** It echoes `embedFraction` /
   `embedDepthCm` (`GroundPlacementHandler.cpp:1001-1002` actors, `:1693-1694` instances); the resolved
   `embedCm` is only on `results[]` rows (`:581`, `:1646`, `:1658`), which `detail` gates and caps at
   256. At `detail: "summary"` the resolved embed is absent.
3. **Default scale is a design question.** A fraction of height sinks a 19 m tree 58 cm and a 0.5 m
   plant 1 cm for the same stated purpose ("do not rest exactly tangent"); `embedDepth` already exists
   as the absolute form.

**Workaround:** pass `embedFraction: 0` with an explicit `embedDepth`, or read `embedCm` from rows at
`detail: "all"`.

**Fix:** add one sentence to both ParamSpecs and to `spatial.ground-placement.md:69`: the height is the
world bounding-box height re-fitted after rotation, so a tilted object embeds deeper than
`meshHeight * scale`. Add a resolved-embed aggregate (e.g. `embedCmMin` / `embedCmMedian` /
`embedCmMax`) to the `seat` echo on both verbs. Decide the default separately (keep `0.02`, or move
to an absolute `embedDepth` with `embedFraction: 0`); flagged for the owner, not asserted.

**Acceptance:** `call({method: "spatial.ground_instances"})` and `spatial.ground_actors` docs state
the embed height is the rotation-inflated world AABB; a batch at `detail: "summary"` returns a
resolved-embed figure in `seat` whose range matches the per-row `embedCm` seen at `detail: "all"`.

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
- `#2-rephrased` `OPEN` developer — Re-checked at PinWright `7230b41d`. The wiki-prose ask is partly done: `docs/wiki-src/spatial.ground-placement.md:104` now explains the rotation-inflated world-AABB height with the 58 cm / 1 cm example (instance section); the general embed paragraph `:69` still does not. Scope narrowed to the remaining ParamSpec sentence (`GroundPlacementHandler.cpp:819-822`, `:1365-1368`), a resolved-embed aggregate in the `seat` echo (`:1001-1002`, `:1693-1694`), and the default decision. All line citations refreshed (old HEAD `6d0e91a3` lines were stale); body trimmed to the template with Fix and Acceptance. Severity unchanged (Low).
