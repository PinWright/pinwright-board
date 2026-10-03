---
id: B-capture-preview-default-camera-vs-light
title: "render.capture_asset_preview's zero-argument camera is fixed on world -X instead of being placed against the preview key light, so the obvious no-args call shoots a grazing-lit face that reads as a black silhouette"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_asset_preview, default-camera, preview-scene, key-light, luminance, false-diagnosis, inverted-normals, docs-placement]
encounters: 1
costly: 1
lastSeen: 2026-08-20T00:00:00Z
rice: [2, 2, 0.8, 2]
priority: 13
---

# The default preview camera ignores where the key light is

With no `location`, `render.capture_asset_preview` places a single still at
`Center + FVector(-Distance, 0, Radius * 0.35)` looking at the centre
(`Source/PinWright/Private/Handlers/Render/RenderHandler.cpp:1342-1350`, pose at `:1345`). That is
always camera azimuth 180° (world -X), whatever the scene's lighting.

The preview key light is not on that side. The engine default rig arrives from azimuth 112.5°,
elevation 40° (`PINWRIGHT_PREVIEW_SCENE_PARAM_DESC`,
`Source/PinWright/Private/Handlers/Render/PreviewViewportCaptureUtils.h:150`; arrival/rotation
conversion `PreviewSceneRig.cpp:172-190`), and it can be moved per call by `previewScene.key` or per
machine by the editor's preview profile. The default camera is therefore ~67.5° off the key: the
face it sees is grazing-lit, and at the pinned exposures callers use it comes back as a black
silhouette. Measured at `ev100 -1` (`docs/wiki-src/visual-review.model-rig.md:59-64`): rook 0.143
default vs 0.285 opposed, large prop 0.174/0.351, sheet model 0.167/0.255, twisted band 0.148/0.255;
the docs attribute three false "inverted normals" reports to this frame
(`docs/wiki-src/model.authoring.md:47`, `visual-review.model-rig.md:66`).

The response now flags the outcome (`subjectRegion.silhouette` + `subjectRegionWarning`,
`SubjectRegionStats.cpp:277,353`) and the `previewScene` description explains it, so the remaining
defect is the pose itself: the verb's default answer is a frame the docs tell every caller to
discard.

The wiki also explains it inconsistently: `render.md:272`, `visual-review.md:12`,
`model.authoring.md:47` and `visual-review.model-rig.md:50` say the default camera sees the "dark" /
"unlit" side, while `visual-review.model-rig.md:52` measures the lit band at camera azimuth 85-130°
and the arrival convention puts 180° inside the lit half (67.5° off the key). The black frame is
grazing light plus the fixed exposure pin (`render.md:262-270`), not a camera on the unlit side.

**Workaround:** pass `location`/`rotation` (camera azimuth 85-95° per
`visual-review.model-rig.md:53`), or a `previewScene.key` rig, or use `exposure: {mode:"auto"}`.

**Fix:** derive the no-args azimuth from the key the capture will actually be lit by: read the
active rig's key arrival (`LightRotationToArrival`, `PreviewSceneRig.cpp:180`, after any
`previewScene` pin is applied) and place the camera a fixed ~25° off it at the existing
bounds-fit distance and elevation; report the chosen default pose in the response. Coordinate with
`E-capture-preview-pose-ignores-bounds-shape`, which changes the same line. Reword the four "dark
side" sentences to "grazing-lit, ~67° off the key".

**Acceptance:** a no-args `render.capture_asset_preview` of `/Engine/BasicShapes/Cube.Cube` under the
default rig returns `subjectRegion.silhouette: false`; with `previewScene.key.azimuth` set to 290 the
default camera moves with it and `silhouette` stays false; explicit `location`/`rotation` still win.

## Related

- `E-capture-preview-pose-ignores-bounds-shape` (OPEN) — the same default-pose line, bounds aspect.
- `B-capture-asset-preview-renders-foliage-black` (IN-REVIEW) — the exposure-pin half of black
  preview subjects.
- `B-capture-preview-decoration-not-suppressible` — same verb.

## History
- `#1-fixed-pose-fixed-rig-decided-apart` `OPEN` reporter — `render.capture_asset_preview`'s zero-argument camera is `Center + FVector(-Distance, 0, Radius*0.35)` looking toward +X (`RenderHandler.cpp:313-317`), an FOV-fit of the bounding sphere with no reference to scene lighting; the scene is an engine `FAdvancedPreviewScene` (`SStaticMeshEditorViewport.cpp:143`) whose light is fixed at `FRotator(-40,-67.5,0)` (`AssetViewerSettings.h:62`, applied `AdvancedPreviewScene.cpp:104`, unoverridden per `Config/DefaultEditor.ini:27-29`) inside a 2000x `EpicQuadPanorama` sky sphere (`AdvancedPreviewScene.cpp:62-86`). Both are fixed and neither was chosen against the other, so every zero-argument capture of every asset lands at the same azimuth to both. Measured at pinned `ev100 -1` (`Docs/wiki-src/visual-review.md:142`, `Docs/wiki-src/model.authoring.md:42`): rook 0.143 default vs 0.285 opposed, driftwood 0.174/0.351, crane 0.167/0.255, mobius 0.148/0.255, and the docs attribute three false "inverted normals" reports to that frame. The reported mechanism is inverted and is corrected here: the light source sits at azimuth 112.5°, so `dot(defaultForward=+X, toLight) = -0.293` puts it *behind* the default camera (front-lit) and *in front of* the opposed one (back-lit); the gap is a frame mean dominated by which half of the panorama backdrop is in shot, not subject shading — which matters because a fix aimed at "turn the camera toward the light" would target the wrong quantity. The wave did not touch this: the only wave commit in `RenderHandler.cpp` is `5e66108e`, adding `hideEditorSprites` to a sibling verb, and `git log -S` on the pose expression last hits `8962f163` (2026-06-23). Workaround (pass explicit `location`/`rotation`) is documented in four overlay pages but not on the method's own page — `Docs/wiki-src/render.md:128-179` never states that `location` is optional, never states the default pose, and carries no lighting warning; the wire description at `RenderHandler.cpp:228` is bare. Fix: derive the default pose from `UAssetViewerSettings::Get()->Profiles[i].DirectionalLightRotation` rather than world -X, move the statement onto `render.md`, and correct the "faces away from the key light" wording in the four pages carrying it.
- `#2-rephrased` `OPEN` developer — Old text cited the pose at `RenderHandler.cpp:303-319` (now `:1342-1350`) and claimed the default pose and lighting warning were documented nowhere on the method's page; the verb's own `previewScene` description now explains the 112.5° key and the black-silhouette misread (`PreviewViewportCaptureUtils.h:150`) and the response flags it via `subjectRegion.silhouette`, so the docs-placement ask is dropped. Rewritten to the remaining defect (fixed world -X pose not placed against the active key) with a rig-relative Fix and an Acceptance, the long engine/backdrop analysis cut, and the doc inconsistency stated as: the camera sits ~67.5° off the key (grazing-lit, inside the lit half per `visual-review.model-rig.md:52`), so the "dark/unlit side" wording in four pages is wrong while the measured black frames stand. Severity unchanged (Medium).
