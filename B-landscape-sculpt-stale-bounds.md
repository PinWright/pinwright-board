---
id: B-landscape-sculpt-stale-bounds
title: "landscape.sculpt / landscape.edit write heights + Flush but never drive the deferred edit-layer update to completion, so an immediate same-turn bounds readback (CachedLocalBox AND live actor.get_bounding_box) still reports Z=0 — the terrain is raised yet the very next RPC reports flat"
status: IN-REVIEW
severity: Medium
category: bug
tags: [stale-bounds-after-height-edit, landscape, sculpt, edit, bounds, collision, deferred-edit-layer-update, same-turn-observability]
encounters: 1
lastSeen: 2026-07-02T12:32:16.9371246+03:00
---

# `landscape.sculpt` / `landscape.edit` return before the edit-layer bounds/collision update runs, so a same-turn bounds readback still reports the pre-sculpt flat Z

`landscape.sculpt` (and the bulk sibling `landscape.edit`) modify the heightmap
through `FLandscapeEditDataInterface::SetHeightData` + `Flush()` and then
`MarkPackageDirty()`, then return `success:true`. On UE 5.7 landscapes are
**edit-layer based**, and `SetHeightData`
(`LandscapeEditInterface.cpp:264-449`) does **not** recompute the component
bounds synchronously — after writing the edit-layer heightmap texels it only
calls `Component->RequestHeightmapUpdate()` (line 438) and
`Component->UpdateDirtyCollisionHeightData(...)` (line 440), i.e. it *schedules*
a deferred layer regeneration. `Flush()`
(`LandscapeEditInterface.cpp`, `FLandscapeTextureDataInterface::Flush`) only
uploads the dirty texture regions to the GPU and frees the texture-data-info
allocations; it does not resolve final heights or touch `CachedLocalBox`.

The deferred pass that *does* fix bounds is
`ALandscape::RegenerateLayersHeightmaps` -> `UpdateForChangedHeightmaps`
(`LandscapeEditLayers.cpp:5037-5074`), which — after the async GPU heightmap
readback — calls `LandscapeComponent->UpdateCachedBounds()` +
`UpdateComponentToWorld()` + `UpdateCollisionData()` per changed component. That
pass runs on a **later editor tick** (driven off
`FLandscapeEditLayerReadback::Tick()`), not inside the RPC. So the RPC returns
before it runs.

Consequence on a normal path (raise a hill, then verify relief **in the very
next RPC**):

1. **A same-turn bounds readback reports flat.** Immediately after a real raise
   (thousands of modified vertices, terrain that *will* be visibly non-flat once
   the layer update ticks), `CachedLocalBox.Max.Z` is still `0`, and the live
   `actor.get_bounding_box` — which computes component bounds via
   `GetActorBounds`, not the cached field — **also** reports `extent Z = 0`,
   because `CachedLocalBox` has not been recomputed yet. A caller trying to
   confirm the terrain has relief in the next call (exactly this task's success
   criterion) gets a false "flat" answer with no error signal.
2. **This is a timing/observability gap, not a permanent inconsistency.** The
   bounds and collision self-correct once the landscape ticks its layer-content
   update (`RegenerateLayersHeightmaps` -> `UpdateForChangedHeightmaps` ->
   `UpdateCachedBounds`), so a re-open / interactive tool touch is *not* required
   to eventually get correct bounds — an editor tick is. The defect is that the
   synchronous RPC returns before that deferred pass runs, so any caller that
   reads bounds back in the same turn sees stale state. (The render/collision
   bounds are momentarily under-sized until the tick, which could briefly affect
   frustum culling, but it resolves on its own.)

The sculpt/edit calls themselves return `success:true` with a correct
`modifiedVertices` count, so the mutation is not silently a no-op — the height
write lands. The defect is that the RPC does not drive the deferred edit-layer
update to completion before returning, so a same-turn bounds readback is stale.

severity rationale: impact=same-turn-stale-readback-on-normal-path (a bounds readback in the RPC immediately after a sculpt reports flat with no signal; self-heals on the next landscape tick, so it is a synchronization/observability gap, not persistent wrong-state) × reach=every-landscape-session (sculpt/edit are the core terrain-shaping verbs and their result is normally verified by reading bounds back) -> Medium

## Culprit source (verbatim)

`Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp`,
`landscape.sculpt` async lambda (lines 623-629):

```cpp
    if (bModified) {
      LandscapeEdit.SetHeightData(MinX, MinY, MaxX, MaxY, HeightData.GetData(), 0, true);
      if (!bSkipFlush) {
        LandscapeEdit.Flush();
      }
      Landscape->MarkPackageDirty();
    }
```

No refresh drives the deferred edit-layer update to completion after the write.
`landscape.edit` has the identical omission (lines 971-979): after
`LandscapeEditWrite.SetHeightData(...)`, `LandscapeEditWrite.Flush()`, and
`Landscape->MarkPackageDirty()`, nothing forces the pending layer regeneration to
run, so `CachedLocalBox` is not recomputed before the RPC returns.

## What it should do

After a modifying `SetHeightData` + `Flush` (when `!bSkipFlush`, i.e. the batch's
final stamp), **drive the deferred edit-layer update to completion synchronously**
before returning, so a same-turn bounds readback reflects the new relief. The
correct mechanism on UE 5.7 is `ALandscape::ForceUpdateLayersContent()`
(`LandscapeEditLayers.cpp:7663` — `LANDSCAPE_API`, public), which calls
`UpdateLayersContent(bWaitForStreaming=true, ..., bFlushRender=true)` and thereby
runs the GPU heightmap readback + `RegenerateLayersHeightmaps` ->
`UpdateForChangedHeightmaps`, whose per-component
`UpdateCachedBounds()` + `UpdateComponentToWorld()` + `UpdateCollisionData()`
(`LandscapeEditLayers.cpp:5058-5063`) is the same path the interactive sculpt
tool uses to flush pending evaluation. This is what refreshes both bounds and
collision.

NOTE — a naive inline `UpdateCachedBounds()`/`MarkRenderStateDirty()`/
`PostEditChange()` right after `Flush()` is the WRONG fix on 5.7's edit-layer
model: `ULandscapeComponent::UpdateCachedBounds`
(`LandscapeEdit.cpp:330`) reads the **non-editing** (merged/final) texture via
`FLandscapeComponentDataInterface CDI(this, MipIndex, /*bWorkOnEditingLayer=*/false)`
(`LandscapeEdit.cpp:395`). Because the sculpt wrote the *edit-layer* texture and
the merge into the final texture is deferred, an immediate `UpdateCachedBounds`
recomputes from the not-yet-regenerated final texture and yields the SAME stale
bounds — a no-op. Only forcing the layer content update (which does the readback +
merge first) makes the recompute see the new heights. (When `skipFlush=true`,
deferring the force-update to the flushing call is fine, mirroring the flush
batching.)

## Verbatim repro (replayed via mcp__pinwright__call)

1. `landscape.create` `{"name":"OracleReplayHillside","componentsX":2,"componentsY":2,"quadsPerComponent":63,"sectionsPerComponent":1,"materialPath":"/Engine/EngineMaterials/WorldGridMaterial.WorldGridMaterial","location":{"x":0,"y":0,"z":0}}`
   -> `success, landscapePath ...PersistentLevel.Landscape_10`
2. `actor.get_component_property` `{"actorName":"OracleReplayHillside","componentName":"LandscapeComponent_0","propertyName":"CachedLocalBox"}`
   -> `{"Min":[0,0,0],"Max":[63,63,0],"IsValid":true}` (baseline flat, Max.Z=0)
3. `landscape.sculpt` `{"landscapeName":"OracleReplayHillside","location":{"x":6300,"y":6300,"z":0},"toolMode":"Raise","brushRadius":4000,"brushFalloff":0.5,"strength":0.6}`
   -> `{"success":true,"toolMode":"Raise","modifiedVertices":3969}` (real raise)
4. `actor.get_component_property` `{...,"propertyName":"CachedLocalBox"}` (re-read)
   -> `{"Min":[0,0,0],"Max":[63,63,0],"IsValid":true}` — **Max.Z still 0** after the raise
5. `actor.get_bounding_box` `{"actorName":"OracleReplayHillside"}`
   -> `{"origin":[8064,8064,0],"extent":[8064,8064,0]}` — **live actor bounds extent Z = 0** too

Both the cached field and the live component-bounds read report the raised
landscape as flat **because the readbacks happen in the next RPC, before the
deferred edit-layer regeneration ticks**. The read verbs are faithful (they
report the real, not-yet-refreshed field / component bounds); the fault is
`landscape.sculpt`/`landscape.edit` returning before the deferred edit-layer
update runs, so a same-turn readback is stale.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live via mcp__pinwright__call. Created a 2x2 landscape (OracleReplayHillside), read `CachedLocalBox.Max=[63,63,0]`, ran `landscape.sculpt` Raise r4000 s0.6 (`modifiedVertices:3969`), re-read `CachedLocalBox.Max=[63,63,0]` (Z unchanged) and `actor.get_bounding_box extent=[8064,8064,0]` (world Z extent 0). Terrain is visibly raised but every bounds readback reports flat, and the landscape's render/collision bounds are left inconsistent with the heightmap. Root cause in `LandscapeHandler.cpp` `landscape.sculpt` (623-629) and `landscape.edit` (971-979): `SetHeightData`+`Flush`+`MarkPackageDirty` with no `UpdateCachedBounds`/`MarkRenderStateDirty`/`RecreateCollision`/`PostEditChange`. `actor.get_component_property` and `actor.get_bounding_box` are faithful (they report the real stale state) — culprit is the sculpt/edit mutators.
- `#2-reword` `OPEN` developer — Rewrote title/body/**Fix:**/severity to match verified UE 5.7 engine source. Correction: on the edit-layer model `SetHeightData` (`LandscapeEditInterface.cpp:264-449`) does NOT update `CachedLocalBox` synchronously — it only `RequestHeightmapUpdate()` (438) + `UpdateDirtyCollisionHeightData` (440), scheduling a DEFERRED regeneration; the fix runs on a later tick via `RegenerateLayersHeightmaps`->`UpdateForChangedHeightmaps`->`UpdateCachedBounds`+`UpdateCollisionData` (`LandscapeEditLayers.cpp:5058-5063`). So the stale Z=0 is a SAME-TURN observability gap that self-heals on the next landscape tick, not permanent inconsistency -> severity High→Medium. The ticket's original proposed fix (inline `UpdateCachedBounds`/`PostEditChange` after `Flush`) is WRONG on 5.7: `UpdateCachedBounds` reads the non-editing/merged texture (`LandscapeEdit.cpp:395`, `bWorkOnEditingLayer=false`), which the deferred merge hasn't populated yet -> recomputes the same stale bounds (no-op). Corrected **Fix:** call `ALandscape::ForceUpdateLayersContent()` (`LandscapeEditLayers.cpp:7663`, `LANDSCAPE_API`) to drive the readback+merge+regeneration synchronously before the RPC returns.
- `#3-fix` `IN-REVIEW` developer — Implemented the reworded fix. `Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp`: after the flushing `SetHeightData`+`Flush()` (only when `!bSkipFlush`), call `Landscape->ForceUpdateLayersContent()` in BOTH `landscape.sculpt` (~line 626) and `landscape.edit` (~line 976). This drives the deferred edit-layer readback + merge + `RegenerateLayersHeightmaps`->`UpdateForChangedHeightmaps`->`UpdateCachedBounds()`/`UpdateCollisionData()` synchronously, so a same-turn bounds readback (`CachedLocalBox`, `actor.get_bounding_box`) reflects the new relief. Deliberately gated on `!bSkipFlush` so batched stamps defer the (potentially costly) force-update to the flushing call, mirroring the existing flush batching. Regression test: `PinWright.landscape.sculpt.RefreshesBoundsAfterRaise` in `Plugins/PinWright/Source/PinWright/Private/Tests/Core/LandscapeSculptRefreshesBoundsTest.cpp` — builds a 2x2/63-quad edit-layer landscape in-code via `landscape.create`, asserts the spawned `ULandscapeComponent.CachedLocalBox.Max.Z` starts flat (~0), runs a real Raise via `landscape.sculpt`, then re-reads `Max.Z` on the same component in the same turn and asserts it rose above baseline (>5, clear of the engine's 1-unit zero-extent flicker guard). Counterfactual: revert the `ForceUpdateLayersContent()` call and `Max.Z` stays at the flat baseline, failing the assertion. Compiles clean (EAContentExamples57Editor Win64 Development).
