---
id: B-capture-preview-decoration-not-suppressible
title: "render.capture_asset_preview cannot suppress any of the four editor decorations it captures — the engine exposes a lever for each one and the plugin passes none of them through"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_asset_preview, decoration, grid, gizmo, backdrop, preview-scene, pass-through-gap, reference-image, docs-placement]
encounters: 2
lastSeen: 2026-09-02T00:00:00Z
---

# Four decorations, four engine levers, zero pass-throughs

`render.capture_asset_preview` captures the live Static Mesh editor viewport, so every frame carries
editor decoration. Its `RPC_PARAMS` block —
`Source/PinWright/Private/Handlers/Render/RenderHandler.cpp:221-237`, twelve parameters
(`assetPath`, `filename`, `target`, `width`, `height`, `location`, `rotation`, `projectionMode`,
`fov`, `orthoWidth`, `exposure`, `closeAfterCapture`) — offers no way to suppress any of it, and the
shared request parser adds none
(`Source/PinWright/Private/Handlers/Render/PreviewViewportCaptureUtils.cpp:947-1010`).

A grep for `showGrid|bDrawAxes|SetFloorVisibility|SetEnvironmentVisibility|EngineShowFlags\.(Grid|Game)|DrawHelper`
over `Source/PinWright/Private/Handlers/Render/` returns only two comment lines
(`PreviewViewportCaptureUtils.cpp:247,257`) explaining why game view is not used. The plugin never
touches a decoration lever anywhere.

## What is drawn, and the lever the engine already exposes

| Element | Created at (engine, UE 5.8.1) | Lever not passed through |
|---|---|---|
| Axis gizmo (bottom-left) | `EditorViewportClient.cpp:531` (`bDrawAxes(true)`), drawn `:5020`/`:5056` → `DrawAxes` `:4135` | `FEditorViewportClient::bDrawAxes` is a **public** member — `EditorViewportClient.h:2042` |
| Grid + coloured axis lines | `StaticMeshEditorViewportClient.cpp:65` (`DrawHelper.bDrawGrid = true`), gated on `EngineShowFlags.Grid` at `EditorComponents.cpp:389` | `FStaticMeshEditorViewportClient::SetShowGrids(bool)` `:1634-1636`, or `FEditorViewportClient::SetShowGrid()` `EditorViewportClient.cpp:6358` |
| Grid floor mesh (`M_Grid` on `Floor_Mesh`, 4×4×1) | `AdvancedPreviewScene.cpp:93-102`, paths `AssetViewerSettings.h:56,58`, visibility `:231` | `FAdvancedPreviewScene::SetFloorVisibility(bVisible, bDirect)` — `AdvancedPreviewScene.h:59` |
| Backdrop (`Sphere_inversenormals` @2000, `M_SkyBox` + `EpicQuadPanorama_CC+EV1`) | `AdvancedPreviewScene.cpp:62-86`, path `AssetViewerSettings.h:54`, visibility `:225` | `FAdvancedPreviewScene::SetEnvironmentVisibility(bVisible, bDirect)` — `AdvancedPreviewScene.h:60` |

Defaults that turn all four on: `AssetViewerSettings.h:41-44`.

So "no flag exists" is true of the plugin and **false of the engine**. This is a pass-through gap,
and the fix is a capture-scoped save/restore of four values, not new rendering.

**Hazard for whoever implements it:** `SetFloorVisibility` / `SetEnvironmentVisibility` must be
called with `bDirect = true`. The `bDirect = false` branch writes
`Profiles[CurrentProfileIndex].bShowFloor` and fires `PostEditChangeProperty` — it permanently
mutates the user's saved preview profile.

## Two corrections to the reported description

1. It is the **grid** that tracks the camera, not the backdrop. `r.Editor.NewLevelGrid` defaults to
   `2` (`EditorComponents.cpp:19-26`), so perspective uses `FGridWidget::DrawNewGrid` (`:146`), which
   anchors the grid plane at `FVector(CameraPos.X, CameraPos.Y, 0)` (`:260-264`, wrapped `:331-341`)
   and colours the axis lines from `GetAxisColors` (`:121-136`) via `UAxisColor`/`VAxisColor`
   (`:302-303`). The backdrop is a **static** sphere at the origin that merely surrounds the camera
   at scale 2000.
2. `Docs/wiki-src/visual-review.md:131` repeats the "backdrop that moves with the camera" wording, so
   the same inaccuracy is in the shipped docs.

## `hideEditorSprites` is not this

Wave commit `5e66108e` added `hideEditorSprites` to sibling verbs and deliberately not to this one:
`Docs/wiki-src/render.md:248` notes the preview scene "contains no actors and therefore no icon
sprites at all", which is correct (`PreviewViewportCaptureUtils.h:24-27`) and orthogonal —
`BillboardSprites` would remove none of the four elements above.

## Documented, but not where an agent lands

`Docs/wiki-src/visual-review.md:131` states it: "`render.capture_asset_preview` is **not**
decoration-free either: it draws an axis gizmo, a grid floor with coloured axis lines, and a
photographic backdrop that moves with the camera." As with the default-camera pose, that statement is
not on the method's own overlay (`Docs/wiki-src/render.md:128-179`), so the generated
`render.capture_asset_preview.md` page says nothing about it.

**Fix:** add one optional parameter (a `decorations: false` / `hideDecorations: true` shape, or a
per-element object) that saves the four values, clears them for the capture and restores them
afterwards, passing `bDirect = true` on both `FAdvancedPreviewScene` setters. Correct the
camera-tracking attribution in `visual-review.md:131` and mirror the limitation onto `render.md`.

## Related

- `B-capture-preview-default-camera-vs-light` — same verb, same docs-placement gap.
- `B-capture-asset-preview-renders-empty` (IN-REVIEW) — mentions the axis-gizmo stub only as a
  symptom of the non-realtime white frame.
- `B-widget-screenshot-preview-includes-chrome` (DONE) — Slate designer chrome on a different verb.

## History
- `#1-four-decorations-no-parameter` `OPEN` reporter — `render.capture_asset_preview` captures the live Static Mesh editor viewport and its `RPC_PARAMS` block (`RenderHandler.cpp:221-237`, twelve parameters) offers no way to suppress any decoration; the shared parser adds none (`PreviewViewportCaptureUtils.cpp:947-1010`) and the plugin never reads or writes a decoration lever anywhere under `Handlers/Render/`. Four elements are drawn: the axis gizmo (`EditorViewportClient.cpp:531`, `bDrawAxes(true)`, drawn `:5020`/`:5056`), the camera-tracking grid with coloured axis lines (`StaticMeshEditorViewportClient.cpp:65`, gated `EditorComponents.cpp:389`, drawn by `FGridWidget::DrawNewGrid` `:146` with its plane anchored to camera XY at `:260-264` and axis colours at `:302-303`), the `M_Grid` floor at 4x scale (`AdvancedPreviewScene.cpp:93-102`) and the `EpicQuadPanorama` sky sphere at scale 2000 (`:62-86`). "No flag exists" holds for the plugin but not the engine: `bDrawAxes` is public (`EditorViewportClient.h:2042`), the grid has `SetShowGrids(bool)` (`StaticMeshEditorViewportClient.cpp:1634`), and floor/backdrop have `SetFloorVisibility`/`SetEnvironmentVisibility` (`AdvancedPreviewScene.h:59-60`) — a pass-through gap, fixable as a capture-scoped save/restore rather than new rendering. Implementation hazard: both `FAdvancedPreviewScene` setters must be called with `bDirect = true`, because the `false` branch writes `Profiles[CurrentProfileIndex].bShowFloor` and fires `PostEditChangeProperty`, permanently mutating the user's saved preview profile. Two corrections to the report: the **grid** tracks the camera, not the backdrop (which is a static origin sphere), and `Docs/wiki-src/visual-review.md:131` carries the same misattribution. `hideEditorSprites` from wave commit `5e66108e` is orthogonal and was deliberately not added here — the preview scene has no actors and therefore no icon sprites (`render.md:248`, `PreviewViewportCaptureUtils.h:24-27`). The limitation is documented at `visual-review.md:131` but not on the method's own overlay (`render.md:128-179`), so the generated method page an agent lands on says nothing.
- `#2-half-landed-orbit-shots-and-a-false-doc` `OPEN` reporter — Second encounter, from a showcase-video pass on `/Game/Maps/Atlantis` (2026-09-02), on a **different verb**, plus a status correction to half this ticket and a new doc defect. **(a) Two of the four levers have landed.** `previewScene` is now a declared parameter on both `render.capture_asset_preview` (`RenderHandler.cpp:341`) and `camera.orbit_shots` (`CameraFrameHandler.cpp:706`); its shape includes `showFloor` and `showEnvironment`, parsed at `PreviewSceneRig.cpp:344-366` and applied via `Advanced->SetFloorVisibility(Pin.bShowFloor, /*bDirect=*/true)` / `SetEnvironmentVisibility(..., true)` at `:703,708`, restored at `:787-788`. The `bDirect = true` hazard `#1` flagged was heeded. So **rows 3 (the `M_Grid` floor) and 4 (the sky backdrop) of this ticket's table are done** and the body is stale on them. **(b) Rows 1 and 2 are untouched.** A repo-wide grep over `Source/PinWright/Private/` for `bDrawAxes`, `SetShowGrid`, `ShowFlags.Grid` and `DrawHelper` returns exactly ONE hit — `Handlers/Render/OrthoTileCaptureUtils.cpp:845`, which only *reports* `Component->ShowFlags.Grid` off a scene-capture component. Nothing in the plugin writes the axis gizmo or the grid, on any verb. **(c) The defect is not verb-specific.** `camera.orbit_shots` with `subject:{kind:"staticMesh", path}` opens and captures the same asset-editor preview viewport, so it carries the same two unsuppressible elements; a fix confined to `render.capture_asset_preview`'s parameter block will not reach it. Reported by the capture agent and NOT re-run by me: frames taken through `camera.orbit_shots` with `previewScene:{showFloor:false, showEnvironment:false}` still showed the world-axis gizmo bottom-left and grid lines. I verified only the mechanism — that no lever for either exists in plugin source. **(d) This ticket's own primary citation is stale**: `RenderHandler.cpp:221-237` is no longer the parameter block. `render.capture_asset_preview` now registers at `RenderHandler.cpp:303` with roughly two dozen parameters spanning `:311-349`. **(e) `hideEditorSprites` correction to `#1`:** still absent from `render.capture_asset_preview` and still orthogonal, but it IS declared on `camera.orbit_shots` (`CameraFrameHandler.cpp:705`), so "UNKNOWN_PARAMS on preview verbs" is a per-verb fact rather than a family rule. **(f) NEW — the shipped wiki now states the opposite of this ticket, on the page an agent lands on.** `Docs/wiki-src/render.md:121` ends: "The warning is emitted for level-viewport captures only; an asset-preview viewport has none of that chrome and no lever to change it." It ships as `Saved/PinWright/wiki/render.a-frame-can-be-contaminated-by-state-this-capture-does-not.md:10`. Both clauses are false: the preview draws the gizmo and grid (`Docs/wiki-src/visual-review.model-rig.md:74`, shipped at `Saved/PinWright/wiki/visual-review.model-rig.md:76`, and `visual-review.md:184` both say so), and `previewScene {showFloor, showEnvironment}` is a shipped lever for two of the four. The two source pages contradict each other in the same wiki. Per a dedup sweep of this board (relayed, not re-derived by me): the sentence is the doc form of `B-game-view-shared-state-no-capture-warning` `#2`'s justification, and that ticket is DONE, so the claim was never re-examined. Fixing `render.md:121` belongs to this ticket, whose Fix section already calls for mirroring the limitation onto `render.md`. **Workaround used 2026-09-02:** `ShowFlag.Grid 0` through `system.console_command` (process-global — see `B-showflag-cvar-override-contaminates-capture`), restored afterwards, plus over-capturing at 1120x1400 and cropping the gizmo off in post. **Severity held at Medium**, not bumped: the reach is two capture verbs rather than an every-session method, and a workaround exists. It is worth noting that the workaround's cost scales with set length — `F-preview-turntable-capture` asks for 240-frame sets, every frame of which needs the same crop — and that the cvar route contaminates every other agent's captures in the shared editor while it is set.
