---
id: B-capture-asset-preview-renders-empty
title: "render.capture_asset_preview returns success but the PNG is empty — preview viewport is not realtime, so the offscreen scene pass never lands in the readback target"
status: IN-REVIEW
severity: High
category: bug
tags: [render, capture_asset_preview, static-mesh, preview, screenshot, blank, silent-success]
---

# render.capture_asset_preview returns success but the PNG is empty — no mesh rendered

`render.capture_asset_preview` is documented as a one-call Static Mesh
"product-shot" capture: it opens the static mesh asset editor, positions the
preview camera, reads the viewport pixels, and writes a PNG. The call returns a
clean success with a plausible result (`captureSource:"staticMeshEditorPreview"`,
`renderer:"sceneViewportReadPixels"`, `mimeType:"image/png"`, correct
`width`/`height`, ~600KB-1.5MB `sizeBytes`) and the PNG is written to disk — but
**the rendered image is empty**: a solid white/blank frame containing only a
faint world-axis gizmo stub in the bottom-left corner. The static mesh is never
visible in the output.

This is not a flat-mesh / out-of-frame framing problem with one weird asset. It
reproduces on the **documentation's own canonical example asset**,
`/Engine/BasicShapes/Cube.Cube`, using the doc's verbatim perspective AND
orthographic example args. A 1x1x1 unit cube captured from `{-300,0,0}` (and from
the doc's `{-300,0,120}` pitch -15) is blank — there is no plausible framing in
which a centered unit cube falls entirely outside the frame at those distances, so
the mesh simply is not being rendered into the captured frame. The call reports
success regardless, making this a **silent success-with-no-effect**: a caller who
trusts the success result and the valid PNG metadata receives an unusable empty
image, and visual verification of a mesh through MCP is impossible via this verb.

Secondary ergonomic nit observed while reproducing: when `location`/`rotation` are
**omitted**, the camera is placed at `{x:0,y:0,z:0}` rotation `{0,0,0}` (echoed in
the result's `cameraLocation`/`cameraRotation`) — i.e. AT the origin, inside the
mesh bounds, which would itself frame nothing even once the render lands. The wiki
documents `{-300,0,120}` only as an *example*, not a default, so this is not a
documented-default contract violation — but a no-args call should still produce a
usable framing rather than a camera buried inside the mesh.

**Workaround:** none through this verb — every camera placement (perspective and
orthographic, near/far, top-down) yields an empty frame. For mesh visual review,
fall back to spawning the mesh into a level and using a level-viewport capture
(`render.capture_open_level` / `editor.screenshot`), which is exactly the
no-spawn workflow this verb was meant to avoid.

**Fix:** the proposed "missing synchronous `Draw()` before `ReadPixels`" diagnosis
is WRONG against current source — `PumpViewport` already calls
`SceneViewport->Invalidate()` + `SceneViewport->Draw()` (`PreviewViewportCaptureUtils.cpp`
L42-47) and is run three times (L180-183) before `ReadPixels` (L186). The real root
cause is that the Static Mesh editor's preview `FEditorViewportClient` is **not
realtime**: `FEditorViewportClient::IsRealtime()` returns false unless a realtime
override (or `RealTimeUntilFrameNumber`) is set, so the manually-driven offscreen
`Draw()` does not reliably composite the 3D scene pass into the slate readback
target — the HUD/canvas pass (the faint axis gizmo) still draws, hence "white frame
+ gizmo stub." The fix is to push a temporary realtime override
(`AddRealtimeOverride(true, ...)` + `RequestRealTimeFrames(N)`) on the preview
viewport client for the duration of the capture, drive the pump loop, then
`RemoveRealtimeOverride` on scope-exit — the engine's intended mechanism for forcing
a non-realtime editor viewport to render offscreen (this is also why the
`render.capture_open_level` level path, whose viewport is typically already realtime,
captures correctly). Independently corroborated by `F-editor-viewport-screenshot #3`,
whose verify recorded the same shared `ReadPixels` path producing a "near-blank"
capture from an un-ticked/non-realtime viewport. Also frame the omitted-`location`
no-args path to the mesh bounds (or a documented `{-300,0,120}`-style offset) instead
of placing the camera at the origin inside the mesh.

## Repro (verbatim)

Doc's canonical perspective example asset, blank result:
```
call render.capture_asset_preview {assetPath:"/Engine/BasicShapes/Cube.Cube",
  filename:"cube_close.png", width:1024, height:768,
  location:{x:-300,y:0,z:0}, rotation:{pitch:0,yaw:0,roll:0},
  projectionMode:"perspective", fov:50}
→ success {path:".../AssetPreview/cube_close.png", width:1024, height:768,
  sizeBytes:670711, cameraLocation:{x:-300,y:0,z:0}, fov:50,
  captureSource:"staticMeshEditorPreview", renderer:"sceneViewportReadPixels",
  mimeType:"image/png"}
PNG on disk = solid white, only a faint axis-gizmo stub bottom-left; no cube.
```

Doc's canonical orthographic example, blank result:
```
call render.capture_asset_preview {assetPath:"/Engine/BasicShapes/Cube.Cube",
  width:1024, height:1024, location:{x:0,y:0,z:0},
  rotation:{pitch:0,yaw:0,roll:0}, projectionMode:"orthographic", orthoWidth:512}
→ success {sizeBytes:1470791, projectionMode:"orthographic", orthoWidth:512, ...}
PNG on disk = blank; no cube.
```

Omitted-camera default lands at origin (not the doc default), also blank:
```
call render.capture_asset_preview {assetPath:"/Game/Global/DemoRoom/Meshes/SM_VizButton.SM_VizButton",
  filename:"viz_button_nanite_replay.png", width:1024, height:768,
  projectionMode:"perspective", fov:50}
→ success {... cameraLocation:{x:0,y:0,z:0}, cameraRotation:{pitch:0,yaw:0,roll:0} ...}
PNG on disk = blank.
```

## History
- `#2-realtime-override-fix` `IN-REVIEW` developer — Reworded the ticket: the original Fix ("missing synchronous Draw before ReadPixels") is demonstrably wrong — `PumpViewport` already calls `SceneViewport->Invalidate()` + `Draw()` (PreviewViewportCaptureUtils.cpp L42-47), run 3x (L180-183) before ReadPixels, so the B-set-camera "missing force-redraw" analogy does not transfer. Root cause: the Static Mesh editor preview `FEditorViewportClient` is NOT realtime by default (`IsRealtime()` is false absent a realtime override / `RealTimeUntilFrameNumber`), so the manually driven offscreen `Draw()` composited only the canvas/HUD pass (the axis gizmo) into the slate readback target — the 3D scene render never landed, hence white-frame + gizmo-stub. Fix in `Source/EditorAutomationRpcGateway/Private/Handlers/Render/PreviewViewportCaptureUtils.cpp` (`CaptureEditorViewportToPng`): push a temporary realtime override via `AddRealtimeOverride(true, ...)` before the pump loop, call `RequestRealTimeFrames(2)` each pump iteration so the scene pass renders, and `RemoveRealtimeOverride(...)` on scope-exit. This is the engine's intended force-render mechanism and also covers the shared `render.capture_open_level` path. Secondary ergonomic fix in `RenderHandler.cpp` (`render.capture_asset_preview`): when `location` is omitted, frame the camera to the static mesh's bounds (pull back along -X by an FOV-fit distance from the bounding-sphere radius, look at center) instead of placing it at the origin inside the mesh — the caller's explicit location/rotation always win. Added `ImageCore`/`ImageWrapper` to Build.cs for the test's PNG decode. Regression test `FRenderCaptureAssetPreviewRendersNonBlankTest` in `Tests/EditorOps/TestRenderHandlers.cpp` captures the engine Cube and asserts the decoded PNG has real color variation (>8 distinct colors) — a blank/uniform frame (the reverted behavior) has ~1 distinct color and fails; headless/no-RHI runs fail with a typed CAPTURE_FAILED/PREVIEW_VIEWPORT_NOT_FOUND/OPEN_FAILED and skip the pixel assertion to avoid false negatives. Not compiled (later phase).
- `#1-initial-repro` `OPEN` reporter — Found while auditing a DemoRoom Nanite-conversion task (seed `render.nanite_rebuild_mesh`, which itself worked). `render.capture_asset_preview` returns clean success with a valid-looking PNG (correct dims, 600KB-1.5MB, `captureSource:"staticMeshEditorPreview"`, `renderer:"sceneViewportReadPixels"`) but the image is an empty white frame with only a faint axis-gizmo stub — the mesh is never rendered. Replay-confirmed on the doc's own example `/Engine/BasicShapes/Cube.Cube` using the doc's verbatim perspective (`location{-300,0,0}`) and orthographic (`location{0,0,0}`, orthoWidth 512) args: both blank. A centered unit cube cannot fall entirely out of frame at those distances, so this is a render-empty bug, not a framing/out-of-bounds issue. Also observed: omitting `location`/`rotation` places the camera at origin `{0,0,0}` (echoed in `cameraLocation`), not the example-documented `{-300,0,120}`. Net: silent success-with-no-effect — caller gets an unusable empty product shot but a success result. Repro args + result shapes recorded above.
