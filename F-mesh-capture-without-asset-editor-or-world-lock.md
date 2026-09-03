---
id: F-mesh-capture-without-asset-editor-or-world-lock
title: "An asset-authoring agent has no way to LOOK at the mesh it just authored: the only two static-mesh capture routes are render.capture_asset_preview, which opens a real asset editor, and render.capture_open_level, which needs the world lock - so on a shared editor every mesh agent but one is blind"
status: IN-REVIEW
severity: High
category: feature
tags: [render, capture_asset_preview, capture_open_level, static-mesh, pwmodel, world-lock, multi-agent, verification, workflow]
encounters: 1
lastSeen: 2026-09-03T04:55:00Z
---

# There is no capture route for a mesh author, and the numbers are not a substitute

## The need

An agent that authors a `.pwmodel` has exactly two ways to see the asset it compiled, and on a
shared editor both are closed to it:

| route | why it is unavailable |
|---|---|
| `render.capture_asset_preview` | Its own page: "It opens the subject's editor/designer". That is the verb `B-capture-asset-preview-no-safe-close-mode` (Critical) is about, and mesh briefs on this project ban opening an asset editor for exactly that reason. |
| `render.capture_open_level` | Needs an actor in the active level, so it needs the world lock. One agent holds that at a time; everyone else waits. |

So on a stream with several mesh agents, at most one can see anything, and only while it owns the
world. The rest author blind and ship on numbers.

## Why numbers are not enough, with this build's evidence

This is not a comfort request. `B-pwmodel-boolean-output-takes-slot-zero` is the proof: on
`Content/FPS/Weapons/Meshes/SM_WPN_AR.pwmodel`, **18,559 of 21,420 triangles - 87% - rendered in the
wrong material while every instrument an author can reach read green.** `health` clean, `bounds`
right, `materialSlotList` right, `unboundSlots` empty, `diagnosticSummary.errors: 0`,
`static_mesh.describe` correct. The defect was found only because somebody looked at a picture.

The same session produced a second case where looking was the only instrument. After the slot fix the
model passed `isClosed true / boundaryEdges 0 / selfIntersections 0 / errors 0`, and a capture taken
by the one agent that held the world lock showed the stock reading as an open frame. Measuring the
silhouette off the baked mesh found it: the comb ran x -23.52..-16.48 while the buttpad's top-front
corner is at x -24.13, leaving a 0.61-wide, 0.7-deep notch in the top line. **Nothing in any response
reports a notch in a silhouette.** It is a closed, manifold, correctly-tagged solid that reads wrong.

Both defects are the same shape: the compile response describes topology and tags, and the failures
that matter to a viewmodel asset are about SHAPE and MATERIAL AS SEEN. There is no overlap.

## What is asked for

A static-mesh capture that opens no asset editor and touches no level. The mesh already exists as a
`UStaticMesh` on disk; rendering one to a PNG needs a transient world or a scene capture component,
not an editor toolkit and not the user's open map. Shape, reusing this surface's existing vocabulary:

    render.capture_static_mesh {
        assetPath, width, height, views | count | location/rotation,
        exposure, viewMode, previewScene, measureCoverage
    }

- No `closeAfterCapture`, because nothing is opened - which also sidesteps
  `B-capture-asset-preview-no-safe-close-mode` entirely rather than fixing its teardown.
- Concurrent-safe: several agents capturing different meshes must not serialise behind one lock.
- `viewMode: front_back_face` matters as much here as elsewhere - an inside-out shell is otherwise
  pixel-identical, and a mesh author is the caller most likely to have produced one.

If a transient-world capture is too large a change, the cheaper version that still unblocks the
workflow is a thumbnail-grade render: `asset.generate_thumbnail` already produces an image for a
`UStaticMesh` without an editor window, and a variant of it that takes a camera and a size, and
returns a path rather than writing an asset thumbnail, would cover most authoring checks.

## Related

- `B-capture-asset-preview-no-safe-close-mode` - why the asset-editor route is banned rather than
  merely slow. This ticket routes around it instead of fixing it.
- `B-pwmodel-boolean-output-takes-slot-zero` - the 87%-wrong-on-green measurement quoted above, and
  the reason its own "report triangles per slot on `model.compile`" ask is necessary but not
  sufficient: per-slot counts would have caught the material fault and would not have caught the
  stock notch.
- `F-isolate-skeletal-mesh-region-arms-only-viewmodel` `#2` - the same world-lock friction one step
  earlier, on the geometry-AUTHORING path rather than the verification path. Together they say an
  asset operation should not need the user's open map at either end.

## History
- `#1-filed` `OPEN` reporter - Filed from the WEAPONS stream on UE 5.8 / EAContentExamples58 after
  finishing `SM_WPN_AR.pwmodel` and `SM_WPN_AR_Magazine.pwmodel` without ever seeing either one. The
  brief banned `render.capture_asset_preview` (asset editor) and the world lock was held elsewhere,
  so every geometric claim in that work - a front iron post and rear aperture placed on a z 9.80
  sight line for lower-1/3 co-witness under the optic, three selector index marks, a serial plate,
  a calibre roll mark, and a rebuilt stock profile - rests on per-triangle read-back with
  `GeometryScript_Materials.get_triangle_material_id` and on arithmetic. That is precisely the
  instrument set `B-pwmodel-boolean-output-takes-slot-zero` proved insufficient in the same session,
  where 87% of the same model shipped in the wrong material with every number green. The one capture
  that was taken - by the agent holding the world lock - immediately found a defect no response
  field carries: a 0.61 x 0.7 notch in the stock's top line, on a mesh that was closed, manifold,
  correctly tagged and passing every gate. Both findings point the same way: for a mesh, looking is
  not a review step that can be deferred to whoever has the lock, it is an instrument with no
  substitute, and it is currently rationed to one agent at a time.
- `#2-transient-mesh-capture` `IN-REVIEW` developer - Added `render.capture_mesh`: each RPC owns one
  transient `FPreviewScene`, shared rig state, mesh component, `USceneCaptureComponent2D`, render
  target, PNG readback and optional subject-coverage reference, reusing them across its shot set
  without opening an asset editor or using the active level world. Registered it as tick-unsafe,
  bounded rendered pixels, and added the GPU-gated cube capture regression test
  `PinWright.render.capture_mesh.TransientCubeHasNonFlatPixels`.

## Fix

The ticket is PARTLY TRUE: the existing `asset.generate_thumbnail` route was already offscreen,
but it did not provide controlled shot sets, `front_back_face`, the shared `previewScene` rig shape,
coverage, or the capture `imageStats` contract. The root cause was that full mesh review was coupled
to editor-owned preview viewports, while the reusable scene-capture probe only exposed float analysis
against a supplied world; the fix adds a colour/PNG production boundary that creates one bare transient
preview world and local rig components per RPC, reuses them across shots, serves both static and skeletal
meshes, supports explicit poses plus `count`/`views:"sides"`, and reports the existing preview-scene,
image-stat, flat-region, and differential-coverage evidence.

Files changed: `Handlers/Render/MeshPreviewCaptureUtils.{h,cpp}`,
`Handlers/Render/SceneCaptureProbeUtils.{h,cpp}`, `Handlers/Render/PreviewSceneRig.{h,cpp}`,
`Handlers/Render/PreviewViewportCaptureUtils.{h,cpp}`, `Handlers/Render/RenderHandler.cpp`,
`Dispatch/SafePoint.cpp`, `Tests/Render/TestMeshPreviewCapture.cpp`, and the render wiki source pages
`render.md`, `render.preview-scene-rig.md`, `render.view-modes.md`, and
`render.capture-exposure.md`. Test id:
`PinWright.render.capture_mesh.TransientCubeHasNonFlatPixels` (not run in this worker wave by rule).
Deliberately unchanged: the existing asset-editor capture and thumbnail contracts, active-world locking,
and skeletal animation playback; animated poses remain the job of `render.capture_animation_preview`.
