---
id: F-ism-per-instance-transforms
title: "No verb reads, writes, or grounds a per-instance transform on an ISM/HISM — and the obvious property-layer write stores silently without moving anything"
status: IN-REVIEW
severity: High
category: feature
tags: [spatial, actor, ism, hism, instanced-static-mesh, per-instance, transform, scatter, ground_actors, find_clear_placement, silent-noop, missing-verb]
encounters: 1
lastSeen: 2026-08-28
---

# A scatter's instances are visible to three subsystems and addressable by none

`F-grounding-holder-not-seatable` made `spatial.verify_grounding` / `spatial.ground_actors` refuse an
ISM/HISM holder with `HOLDER_NOT_SEATABLE` and point callers at "the per-instance path". There is no
per-instance path. The refusal text itself now says so out loud
(`GroundPlacementUtils.cpp:509-528`): *"a non-foliage ISM/HISM has no per-instance write verb here and
has to be re-scattered by whatever built it."*

**Zero calls to `GetInstanceTransform` exist anywhere in the tree, tests included.** Every non-test
`AddInstance` / `UpdateInstanceTransform` / `RemoveInstance` / `BatchUpdateInstancesTransforms` call
site is absent; the only `AddInstance` hits outside `Private/Tests/**` are `FFoliageInfo::AddInstance`
in the foliage handler, which is a different API on a different object.

What the plugin *does* have is three independent non-test sites that see instances and can only look:

- `SpatialTraceUtils.cpp:420-469` — resolves the blocking instance index via `GetItemIndex()`, falling
  back to `GetInstancesOverlappingBox`. Read-only index query; never reads a transform.
- `MeasureHandler.cpp:1228-1232` — `spatial.find_clear_placement` reports `instanceIndices[]` and
  `GetInstanceCount()`. Its own summary calls instance attribution "the half no name-based occupancy
  input here can express". It names the instance in your way and offers no way to move it.
- `GroundPlacementUtils.cpp:485-506` — `FindInstancedHolder`, read-only detection used **only to
  refuse**.
- (`LevelAuditUtils.cpp:1905-1922` counts instances purely to emit a caveat saying they were not
  examined; `PCGGenerateReadback.h:154-163` counts PCG-managed ISMs.)

## `foliage.*` does not cover this, and fails dangerously when pointed at it

All six foliage verbs resolve through `GetOrCreateFoliageActorForWorldSafe`
(`FoliageHandler.cpp:39-72`), hard-typed to `AInstancedFoliageActor`, and operate on the
`AInstancedFoliageActor` + `UFoliageType` + `FFoliageInfo` triple. None takes an actor or component
parameter — `actorName` hits the dispatcher's `UNKNOWN_PARAMS` gate before the handler body runs. So a
plain BP actor carrying a HISM, a PCG-managed ISM, or a construction-script scatter is invisible to all
of them. Two of the failure modes are silent:

- `foliage.get_instances` with no IFA in the world returns **`success:true` with an empty
  `instances[]`** (`FoliageHandler.cpp:410-416`) — the scatter is simply not seen.
- `foliage.add_instances` pointed at the holder's static mesh **auto-creates a new `UFoliageType` at
  `/Game/Foliage/Auto_<MeshName>`** (`:850-860`) and writes into an IFA (`:874-889`), reporting
  `success:true`. The HISM is untouched and the level now has a second, parallel scatter.

## What a caller must do today, and why each route fails

**Read — works, badly.** `property.get` takes `omitOversized` defaulting to **false**
(`UtilityPropertyHandler.cpp:1484/1504`), so the default call returns the whole `PerInstanceSMData`
array; `container.array.get` reaches one element by index. Both hand back
`FInstancedStaticMeshInstanceData::Transform` as a raw reflected **`FMatrix` in component-local space**
(`InstancedStaticMeshComponent.h:44-45`). The caller decomposes the matrix client-side and composes it
with the component's world transform to get anything comparable to a `spatial.raycast` hit. Meanwhile
`asset.dump` / `blueprint.*` / SCS dumps elide the same field unconditionally
(`PropertyExport.cpp:588-594, 1466-1472`), so the cached asset dumps show only a count.

**Write via the property layer — stores, reports success, moves nothing.** This is the load-bearing
failure. `PerInstanceSMData` is *not* blocked on write: `IsKnownOversizedProperty` has five call sites
and all five are export paths (`PropertyExport.cpp:554, 588, 1419, 1466`,
`UtilityPropertyHandler.cpp:983-986`, `PinWright_SCSHandlers.cpp:264-267`); nothing in
`PropertyImport.cpp` consults it. Path resolution supports `PerInstanceSMData[3].Transform`
(`PropertyInspection.cpp:111-201`) and the struct-inner write branch handles it
(`PropertyImport.cpp:1072-1117`). So `property.set` / `container.array.set` **succeed**, and a readback
**confirms the new matrix** — while the world does not change:

- The only notification PinWright ever emits is the **non-chain** `PostEditChangeProperty`
  (`PropertyChangeNotify.h:88-108`), and `UInstancedStaticMeshComponent` has **no
  `PostEditChangeProperty` override** — all its instance handling lives in
  `PostEditChangeChainProperty` (`InstancedStaticMesh.cpp:5577-5650`). `UObject` calls chain→non-chain,
  never the reverse, so the handling is unreachable. Skipped: `InvalidateCachedBounds()`,
  `FullNavigationUpdate()`, `PrimitiveInstanceDataManager.TransformsChangedAll()` (`:5622-5626`), and
  for HISM `BuildTreeIfOutdated` (`HierarchicalInstancedStaticMesh.cpp:2110-2120`).
- Synthesising the chain form is **not** an option and must not be attempted: ISM dereferences
  `PropertyChain.GetActiveMemberNode()` with no null check (`InstancedStaticMesh.cpp:5636`), which
  crashes the editor. This is already documented at `PropertyChangeNotify.h:34-40` and is exactly why
  the plugin emits only the non-chain form.
- **`MarkRenderStateDirty()` does not substitute.** UE 5.8 gates instance-buffer uploads on change
  *tracking*, not proxy recreation: `FPrimitiveInstanceDataManager::FlushChanges` computes
  `bInstancesChanged = HasAnyInstanceChanges()` and early-returns when nothing was marked
  (`ISMInstanceDataManager.cpp:577, 605-608`). A raw reflection store marks nothing, and the
  instance-data proxy survives render-state recreation (`ISMInstanceDataManager.cpp:885-913`), so the
  GPU buffers keep the old transforms. Physics bodies and navigation are likewise never told.

**Write via `object.call_function` — works, and nothing says so.** `UpdateInstanceTransform` is
`UFUNCTION(BlueprintCallable)` (`InstancedStaticMeshComponent.h:374-375`), so
`object.call_function {function:"UpdateInstanceTransform"}` does the tracker + physics + nav work
itself. This is a reflection escape hatch found only by reading engine source, costs one RPC per
instance, and is contradicted by the `HOLDER_NOT_SEATABLE` text that tells callers to re-scatter
instead.

**Grounding — no route at all.** Even with the escape hatch, seating one instance means reimplementing
`MeasureContact`'s footprint-column solve client-side: derive the instance's world bounds, sample a
grid, trace ground per column under the same `surface` filter, aggregate by percentile, apply embed,
re-measure. None of that is reachable from outside the plugin.

## Proposed verb shape — three verbs, not one

Splitting read / write / solve mirrors the pairs the plugin already has
(`foliage.get_instances` ↔ `foliage.add_instances`; `spatial.verify_grounding` ↔
`spatial.ground_actors`).

1. **`actor.get_instances`** — `{actorName, component?, indices?, limit?, offset?, space:"world"|"local"}`
   → `[{index, location, rotation, scale}]` as decomposed transforms, plus `instanceCount`. Carries
   scale from day one (`E-foliage-get-instances-drops-scale` is the same omission on the foliage side).
   Sits next to `actor.get_components`.
2. **`actor.set_instance_transforms`** — `{actorName, component?, instances:[{index, location?,
   rotation?, scale?}], space, expectedCount?}`. Batch by construction: a scatter fix is N instances,
   and one RPC per instance is the current cost. Echoes each instance's pre-move transform in
   `movedInstances[]`, the convention `spatial.ground_actors` already documents as "that is the undo".
3. **`spatial.ground_instances`** — `{actorName, component?, indices?, surface (REQUIRED, same schema
   as `ground_actors`), samples, seatPercentile, embed, apply:true}` → per-instance
   `{index, placed, reason, before, after, clearance}`.

`ground_instances` should **not** be split into a verify/apply pair. That split exists on the actor
side because a name-pattern selector on a mutating verb needs `expectedMatches` guarding it; instances
are addressed by explicit index against one named component, so there is no pattern hazard and an
`apply:false` dry run covers the verify half.

**Fix:**

*Reuse, concretely.*
- `GroundPlacement::FindInstancedHolder` (`GroundPlacementUtils.cpp:474-507`) resolves the target
  component when `component` is omitted — it already picks the component carrying the most instances.
- `GroundPlacement::FGroundSurfaceSpec` + `ParseSurfaceJson` / `ParseSurfacePreset`
  (`GroundPlacementUtils.h:121-180`) take the `surface` param verbatim, so instance grounding filters
  ground identically to `ground_actors`.
- `GroundPlacement::MeasureContact` (`GroundPlacementUtils.cpp:684`) is **not** reusable as-is: it
  takes `AActor*`, derives the footprint from `Actor->GetActorBounds` (`:715`) and ignores `Actor` in
  the trace (`:740`). Extract everything below the bounds call into
  `MeasureContactForBounds(World, const FBox& WorldBounds, const TArray<AActor*>& Ignore, Surface,
  GridSize, Inset, UndersideModel, Thresholds, OutColumns)` and have `MeasureContact` delegate to it.
  Per instance, `WorldBounds` = the static mesh's bounds under (instance local transform × component
  world transform). This is the one seam that needs cutting; everything else already fits.
- `AggregateColumns`, `EvaluateContact`, `FGroundColumn`, `FContactThresholds`,
  `FGroundSeatConfig::ResolveEmbedCm`, `SeatStatusToString` — all column/bounds-level, reusable
  unchanged.
- `SpatialTraceUtils::TraceGroundBelow` (`SpatialTraceUtils.h:218`) already takes an `FBox`, so
  per-instance ground probing needs no new trace code.
- `spatial.find_clear_placement` (`MeasureHandler.cpp:638`) needs no change — it is the caller-side
  pairing. Once instances are addressable its instance attribution becomes actionable rather than
  informational: search a clear pose, then write it to the named instance.

*Writing, transactions, render state.*
- Call `UInstancedStaticMeshComponent::UpdateInstanceTransform` (or
  `BatchUpdateInstancesTransforms`) directly. **Never** route through the reflection property layer,
  and **never** synthesise a `PostEditChangeChainProperty` — see the crash at
  `InstancedStaticMesh.cpp:5636`. `UpdateInstanceTransform` already does `Modify()`,
  `InvalidateCachedBounds()`, the write, `PrimitiveInstanceDataManager.TransformChanged(Index)`,
  `UpdateInstanceBodyTransform` under `bPhysicsStateCreated`, and the navigation update.
- Batch: pass `bMarkRenderStateDirty=false` on every call and dirty **once** at the end. The house
  pattern is `Modify() -> mutate -> MarkRenderStateDirty() -> MarkPackageDirty()`
  (`SpawnMaterialUtils.h:41`).
- **HISM needs the cluster tree rebuilt** — moved instances leave it stale, and culling plus
  per-instance collision read it. Use `BuildTreeIfOutdated(/*Async*/false, /*ForceUpdate*/true)` under
  an `FApp::CanEverRender()` guard: the exact form already used after a mesh swap at
  `ComponentAssetPropertyWrite.cpp:156-163`, which mirrors HISM's own chain-property branch.
- Undo: wrap in `FScopedTransaction` per the house pattern, but note the cost — `Modify()` on an ISM
  serialises the whole `PerInstanceSMData` array into the transaction buffer, so a 10k-instance
  scatter produces a large undo record per call. Either offer an `undo:false` opt-out or lean on the
  echoed `movedInstances[]` pre-move transforms, which `ground_actors` already treats as the undo of
  record.

*Follow-on.* Update `DescribeInstancedHolder` (`GroundPlacementUtils.cpp:509-528`) to route callers at
the new verbs instead of "re-scattered by whatever built it".

*Possible split.* The silent `property.set` / `container.array.set` store on `PerInstanceSMData` is a
defect in its own right and could be filed separately as a bug — either refuse the field on write with
a typed error naming `actor.set_instance_transforms`, or add it to a write-side block list. It is
recorded here because it is the reason the obvious route fails, and because a refusal is only
defensible once there is a verb to name.

## History
- `#1-no-per-instance-path` `OPEN` reporter — Found while reviewing `F-grounding-holder-not-seatable`
  (IN-REVIEW), whose refusal points at a per-instance path that does not exist. Confirmed absent:
  zero `GetInstanceTransform` / `UpdateInstanceTransform` / `SetInstanceTransform` /
  `BatchUpdateInstancesTransforms` / `RemoveInstance` call sites anywhere in Source, tests included;
  every non-test ISM/HISM touch is read-only counting or detection (`SpatialTraceUtils.cpp:420-469`,
  `MeasureHandler.cpp:1228-1232`, `GroundPlacementUtils.cpp:485-506`, `LevelAuditUtils.cpp:1905-1922`,
  `PCGGenerateReadback.h:154-163`). `foliage.*` is hard-typed to `AInstancedFoliageActor`
  (`FoliageHandler.cpp:39-72`) and cannot be pointed at a plain holder; `foliage.get_instances` returns
  an empty success and `foliage.add_instances` silently builds a parallel scatter. Reading works only
  as a raw component-local `FMatrix` through `property.get` / `container.array.get`; writing through
  the property layer stores and reports success while the render tracker, physics bodies, nav and HISM
  tree are never told (`ISMInstanceDataManager.cpp:605-608`, `InstancedStaticMesh.cpp:5577-5650`); the
  only working write is an undocumented `object.call_function` on the BlueprintCallable
  `UpdateInstanceTransform`, and grounding has no route at all. Proposed `actor.get_instances`,
  `actor.set_instance_transforms`, `spatial.ground_instances` reusing `FindInstancedHolder`, the
  `surface` spec, `TraceGroundBelow`, and a bounds-driven `MeasureContact` extraction.
- `#2-three-verbs-shipped` `IN-REVIEW` developer — All three verbs landed, in the proposed shape.
  `actor.get_instances` `{actorName, component?, indices?, limit?, offset?, space:world|local}` ->
  `instances[{index, location, rotation, scale}]` + `instanceCount`; scale carried.
  `actor.set_instance_transforms` `{actorName, component?, instances[{index, location?, rotation?,
  scale?}], space, expectedCount?}` — all-or-nothing pre-flight (one stale index, one duplicated
  index, or an `expectedCount` disagreeing with `GetInstanceCount()` refuses the whole batch before
  the first write), `movedInstances[]` echoes each pre-write transform, and `updated` is derived
  from a post-write READBACK rather than from the call returning true.
  `spatial.ground_instances` `{actorName, component?, indices?, surface (required), samples,
  seatPercentile, embed*, apply}` — not split into a verify/apply pair, `apply:false` is the dry
  run and reports `proposedDeltaZCm` / `status:"dry_run"` with the identical solve and thresholds.
  Extracted `MeasureContactForBounds(World, FBox, UndersideGeometry, ExtraIgnoreActors, Surface,
  Grid, Inset, UndersideModel, Thresholds, OutColumns)` as proposed; `MeasureContact` is now a thin
  wrapper over it (the holder probe moved after the delegate call and re-evaluates only for a
  holder). Two additions the ticket's signature did not anticipate: a nullable `UndersideGeometry`
  actor, because the per-column underside probe needs one and an instance has none — so an instance
  is measured with `EUndersideModel::BoundsPlane` and the response says so, since
  `UInstancedStaticMeshComponent::LineTraceComponent` answers from every instance body at once
  (`InstancedStaticMesh.cpp:5442-5444`) and cannot attribute a hit; and `ExtraIgnoreActors`, since
  the function cannot know what it is measuring. New `GroundPlacement::SeatInstance` +
  `FGroundInstanceSeatResult` (composes `FGroundSeatResult`), new `EGroundSeatStatus::DryRun`, and
  `GroundStatusForMeasurementFailure` extracted so the actor and instance paths cannot classify the
  same measurement differently. Writes go through `UpdateInstanceTransform` only, with
  `bMarkRenderStateDirty=false` per call and one `InstancedMeshUtils::FinishInstanceWrites` per
  batch (`MarkRenderStateDirty` -> HISM `BuildTreeIfOutdated(false,true)` under
  `FApp::CanEverRender()` -> `MarkPackageDirty`) — needed because HISM's own override queues an
  ASYNC non-forced rebuild per moved instance. **Transaction decision: no `FScopedTransaction`**;
  `Modify()` on an ISM serialises the whole `PerInstanceSMData` array per call, so a batch would
  cost O(written x total) of undo record, and the echoed `movedInstances[]` pre-write transforms are
  the undo of record (they round-trip straight back through `actor.set_instance_transforms`) —
  rationale recorded in `Handlers/Actor/InstancedMeshUtils.h`. `DescribeInstancedHolder`'s
  `HOLDER_NOT_SEATABLE` text and `Docs/wiki-src/spatial.md` now route callers at the new verbs
  instead of "re-scattered by whatever built it". Files: new
  `Handlers/Actor/InstancedMeshHandler.cpp`, new `Handlers/Actor/InstancedMeshUtils.h`, edited
  `Handlers/Spatial/GroundPlacementUtils.{h,cpp}`, `Handlers/Spatial/GroundPlacementHandler.cpp`,
  `Handlers/ErrorCodes.h` (+`INSTANCE_INDEX_OUT_OF_RANGE`, +`NO_INSTANCED_COMPONENT`),
  `Docs/wiki-src/spatial.md`, `Tests/Spatial/TestGroundPlacement.cpp`. Four new tests on the
  existing HISM fixture, every transform assertion re-read from the component rather than from the
  response: `actor.get_instances.ReadsDecomposedWorldAndLocalTransforms`,
  `actor.set_instance_transforms.WriteReachesTheComponent`,
  `spatial.ground_instances.SeatsEachInstanceWithoutMovingTheHolder`,
  `spatial.ground_instances.DryRunSolvesAndWritesNothing`. NOT COMPILED AND NOT RUN — the wave
  forbade building; a tester must compile and run `PinWright.spatial.*` + `PinWright.actor.*`.
