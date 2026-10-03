---
id: B-capture-preview-decoration-not-suppressible
title: "Asset-editor preview captures cannot suppress the world-axis gizmo or the grid: no capture verb passes bDrawAxes or the Grid show flag through, on render.capture_asset_preview or on the asset-subject branch of camera.orbit_shots and its siblings"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_asset_preview, decoration, grid, gizmo, backdrop, preview-scene, pass-through-gap, reference-image, docs-placement]
encounters: 2
costly: 1
lastSeen: 2026-09-02T00:00:00Z
rice: [2, 2, 1, 2]
priority: 17
---

# Asset-editor preview captures always carry the axis gizmo and the grid

Every verb that photographs an asset editor's preview viewport draws two editor decorations into
every frame, and no parameter removes either:

- the world-axis gizmo, bottom-left (`FEditorViewportClient::bDrawAxes`, a public member,
  `EditorViewportClient.h:2042` in UE 5.8);
- the grid with coloured axis lines (`EngineShowFlags.Grid` on the preview client; the Static Mesh
  editor also exposes `FStaticMeshEditorViewportClient::SetShowGrids(bool)`).

Affected verbs (all route through the shared preview-viewport capture in
`Source/PinWright/Private/Handlers/Render/PreviewViewportCaptureUtils.cpp`):
`render.capture_asset_preview` (`Source/PinWright/Private/Handlers/Render/RenderHandler.cpp:623`,
params `:624-668`), `render.capture_animation_preview` (`AnimationPreviewCaptureHandler.cpp:195`),
and the asset-subject branch of `camera.orbit_shots` (`CameraFrameHandler.cpp:684`),
`camera.frame_actor` (`CameraFrameHandler.cpp:235`) and `render.capture_annotated`
(`AnnotatedCaptureHandler.cpp:187`).

The floor and the backdrop are already suppressible: `previewScene {showFloor, showEnvironment}`
applies `SetFloorVisibility` / `SetEnvironmentVisibility` with `bDirect = true`
(`Source/PinWright/Private/Handlers/Render/PreviewSceneRig.cpp:1015,1020`, restored `:1085-1086`).
The gizmo and grid have no equivalent: a grep of `Source/` for `bDrawAxes`, `SetShowGrid`,
`ShowFlags.Grid`, `DrawHelper` finds only `OrthoTileCaptureUtils.cpp:845` (reports a scene-capture
component's flag) and a test log line (`Tests/Render/TestPoseListCaptureStability.cpp:487`), both
reads. The limitation is documented correctly at `docs/wiki-src/render.md:145`, but that paragraph's
citations are stale (`RenderHandler.cpp:341` is now `:661`, `CameraFrameHandler.cpp:706` is now
`:712`) and it does not mention the decoration-free route below.

**Workaround:** for a Static or Skeletal Mesh use `render.capture_mesh`
(`RenderHandler.cpp:347`): it draws through a `USceneCaptureComponent2D` in a private
`FPreviewScene` (`docs/wiki-src/render.md:328`) with game-base show flags
(`OrthoTileCaptureUtils.cpp:165-171`, used at `MeshPreviewCaptureUtils.cpp:353`), so no editor
viewport chrome is drawn (source-read, not live-verified). Otherwise `ShowFlag.Grid 0` through
`system.console_command` (process-global, contaminates other agents' captures; set back afterwards)
and crop the gizmo off in post.

**Fix:** in the shared preview capture path, beside the existing sprite scope
(`FScopedEditorSpriteSuppression`, `PreviewViewportCaptureUtils.cpp:2359`), add a capture-scoped
save/clear/restore of `bDrawAxes` and `EngineShowFlags.Grid` on the client being captured, driven by
one optional parameter (e.g. `previewScene.showAxes` / `previewScene.showGrid`, default unchanged)
declared on every verb listed above, and report the applied values in the response. In
`render.md:145`, refresh the two citations and point mesh callers at `render.capture_mesh`.

**Acceptance:** `render.capture_asset_preview` and `camera.orbit_shots {subject:{kind:"staticMesh"}}`
with the new flags off return frames with no gizmo in the bottom-left corner and no grid lines
(visible in the PNG); the viewport's `bDrawAxes` and Grid flag read the same before and after the
call; omitting the flags leaves today's pixels unchanged.

## Related

- `B-capture-preview-default-camera-vs-light` — same verb.
- `B-orbit-shots-sprites-inert-on-asset-subject` — same verb family, `hideEditorSprites` (billboard
  icons), which is orthogonal to these two elements.
- `B-showflag-cvar-override-contaminates-capture` (DONE) — the process-global console route the workaround relies on.

## History
- `#1-four-decorations-no-parameter` `OPEN` reporter — `render.capture_asset_preview` captures the live Static Mesh editor viewport and its `RPC_PARAMS` block (`RenderHandler.cpp:221-237`, twelve parameters) offers no way to suppress any decoration; the shared parser adds none (`PreviewViewportCaptureUtils.cpp:947-1010`) and the plugin never reads or writes a decoration lever anywhere under `Handlers/Render/`. Four elements are drawn: the axis gizmo (`EditorViewportClient.cpp:531`, `bDrawAxes(true)`, drawn `:5020`/`:5056`), the camera-tracking grid with coloured axis lines (`StaticMeshEditorViewportClient.cpp:65`, gated `EditorComponents.cpp:389`, drawn by `FGridWidget::DrawNewGrid` `:146` with its plane anchored to camera XY at `:260-264` and axis colours at `:302-303`), the `M_Grid` floor at 4x scale (`AdvancedPreviewScene.cpp:93-102`) and the `EpicQuadPanorama` sky sphere at scale 2000 (`:62-86`). "No flag exists" holds for the plugin but not the engine: `bDrawAxes` is public (`EditorViewportClient.h:2042`), the grid has `SetShowGrids(bool)` (`StaticMeshEditorViewportClient.cpp:1634`), and floor/backdrop have `SetFloorVisibility`/`SetEnvironmentVisibility` (`AdvancedPreviewScene.h:59-60`) — a pass-through gap, fixable as a capture-scoped save/restore rather than new rendering. Implementation hazard: both `FAdvancedPreviewScene` setters must be called with `bDirect = true`, because the `false` branch writes `Profiles[CurrentProfileIndex].bShowFloor` and fires `PostEditChangeProperty`, permanently mutating the user's saved preview profile. Two corrections to the report: the **grid** tracks the camera, not the backdrop (which is a static origin sphere), and `Docs/wiki-src/visual-review.md:131` carries the same misattribution. `hideEditorSprites` from wave commit `5e66108e` is orthogonal and was deliberately not added here — the preview scene has no actors and therefore no icon sprites (`render.md:248`, `PreviewViewportCaptureUtils.h:24-27`). The limitation is documented at `visual-review.md:131` but not on the method's own overlay (`render.md:128-179`), so the generated method page an agent lands on says nothing.
- `#2-half-landed-orbit-shots-and-a-false-doc` `OPEN` reporter — Second encounter, from a showcase-video pass on `/Game/Maps/Atlantis` (2026-09-02), on a **different verb**, plus a status correction to half this ticket and a new doc defect. **(a) Two of the four levers have landed.** `previewScene` is now a declared parameter on both `render.capture_asset_preview` (`RenderHandler.cpp:341`) and `camera.orbit_shots` (`CameraFrameHandler.cpp:706`); its shape includes `showFloor` and `showEnvironment`, parsed at `PreviewSceneRig.cpp:344-366` and applied via `Advanced->SetFloorVisibility(Pin.bShowFloor, /*bDirect=*/true)` / `SetEnvironmentVisibility(..., true)` at `:703,708`, restored at `:787-788`. The `bDirect = true` hazard `#1` flagged was heeded. So **rows 3 (the `M_Grid` floor) and 4 (the sky backdrop) of this ticket's table are done** and the body is stale on them. **(b) Rows 1 and 2 are untouched.** A repo-wide grep over `Source/PinWright/Private/` for `bDrawAxes`, `SetShowGrid`, `ShowFlags.Grid` and `DrawHelper` returns exactly ONE hit — `Handlers/Render/OrthoTileCaptureUtils.cpp:845`, which only *reports* `Component->ShowFlags.Grid` off a scene-capture component. Nothing in the plugin writes the axis gizmo or the grid, on any verb. **(c) The defect is not verb-specific.** `camera.orbit_shots` with `subject:{kind:"staticMesh", path}` opens and captures the same asset-editor preview viewport, so it carries the same two unsuppressible elements; a fix confined to `render.capture_asset_preview`'s parameter block will not reach it. Reported by the capture agent and NOT re-run by me: frames taken through `camera.orbit_shots` with `previewScene:{showFloor:false, showEnvironment:false}` still showed the world-axis gizmo bottom-left and grid lines. I verified only the mechanism — that no lever for either exists in plugin source. **(d) This ticket's own primary citation is stale**: `RenderHandler.cpp:221-237` is no longer the parameter block. `render.capture_asset_preview` now registers at `RenderHandler.cpp:303` with roughly two dozen parameters spanning `:311-349`. **(e) `hideEditorSprites` correction to `#1`:** still absent from `render.capture_asset_preview` and still orthogonal, but it IS declared on `camera.orbit_shots` (`CameraFrameHandler.cpp:705`), so "UNKNOWN_PARAMS on preview verbs" is a per-verb fact rather than a family rule. **(f) NEW — the shipped wiki now states the opposite of this ticket, on the page an agent lands on.** `Docs/wiki-src/render.md:121` ends: "The warning is emitted for level-viewport captures only; an asset-preview viewport has none of that chrome and no lever to change it." It ships as `Saved/PinWright/wiki/render.a-frame-can-be-contaminated-by-state-this-capture-does-not.md:10`. Both clauses are false: the preview draws the gizmo and grid (`Docs/wiki-src/visual-review.model-rig.md:74`, shipped at `Saved/PinWright/wiki/visual-review.model-rig.md:76`, and `visual-review.md:184` both say so), and `previewScene {showFloor, showEnvironment}` is a shipped lever for two of the four. The two source pages contradict each other in the same wiki. Per a dedup sweep of this board (relayed, not re-derived by me): the sentence is the doc form of `B-game-view-shared-state-no-capture-warning` `#2`'s justification, and that ticket is DONE, so the claim was never re-examined. Fixing `render.md:121` belongs to this ticket, whose Fix section already calls for mirroring the limitation onto `render.md`. **Workaround used 2026-09-02:** `ShowFlag.Grid 0` through `system.console_command` (process-global — see `B-showflag-cvar-override-contaminates-capture`), restored afterwards, plus over-capturing at 1120x1400 and cropping the gizmo off in post. **Severity held at Medium**, not bumped: the reach is two capture verbs rather than an every-session method, and a workaround exists. It is worth noting that the workaround's cost scales with set length — `F-preview-turntable-capture` asks for 240-frame sets, every frame of which needs the same crop — and that the cvar route contaminates every other agent's captures in the shared editor while it is set.
- `#3-rephrased` `OPEN` developer — Old text described four unsuppressible decorations; the floor and backdrop are now suppressible through `previewScene {showFloor, showEnvironment}` (`PreviewSceneRig.cpp:1015,1020`, `bDirect = true`), and the docs asks it carried are done (`render.md:145` now states the gizmo/grid limitation; the `visual-review.md` "moves with the camera" wording is gone). Narrowed to the axis gizmo and grid, widened from two verbs to every verb that captures an asset-editor preview (`render.capture_asset_preview`, `render.capture_animation_preview`, asset subjects of `camera.orbit_shots` / `camera.frame_actor` / `render.capture_annotated`), dropped the engine-line table and stale `RenderHandler.cpp:221-237` citation, added `render.capture_mesh` as a decoration-free workaround for mesh subjects and the stale citations in `render.md:145`, added Acceptance. Severity unchanged (Medium). RICE effort E 1->2: the widened fix is a shared capture-scope guard plus a parameter declared on five verbs, with tests.
