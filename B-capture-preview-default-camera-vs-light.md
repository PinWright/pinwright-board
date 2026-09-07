---
id: B-capture-preview-default-camera-vs-light
title: "render.capture_asset_preview's zero-argument camera is a fixed world-axis pose chosen with no reference to the fixed preview lighting rig — the frame it returns has been misread as inverted normals three times"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_asset_preview, default-camera, preview-scene, key-light, luminance, false-diagnosis, inverted-normals, docs-placement]
encounters: 1
costly: 1
lastSeen: 2026-08-20T00:00:00Z
---

# A fixed camera pose against a fixed light rig, decided independently

The whole no-args camera path is
`Source/PinWright/Private/Handlers/Render/RenderHandler.cpp:303-319`:

```cpp
const FVector CameraLocation = Center + FVector(-Distance, 0.0f, Radius * 0.35f);   // :313
Request.Rotation = (Center - CameraLocation).Rotation();                            // :317
```

`Distance` is an FOV-fit of the bounding sphere (`:305-311`). The pose is therefore always **−X,
looking toward +X**, pitched slightly down, and nothing in that block reads the preview scene's
lighting. `ApplyCaptureCamera` (`Source/PinWright/Private/Handlers/Render/PreviewViewportCaptureUtils.cpp:698-728`)
applies it verbatim at `:724`/`:727`.

The scene is an engine `FAdvancedPreviewScene`
(`C:\UE_5.8\Engine\Source\Editor\StaticMeshEditor\Private\SStaticMeshEditorViewport.cpp:143`) whose
directional light is fixed at `FRotator(-40, -67.5, 0)`
(`C:\UE_5.8\Engine\Source\Editor\AdvancedPreviewScene\Public\AssetViewerSettings.h:62`, applied at
`AdvancedPreviewScene.cpp:104`), and whose environment is a 2000×-scaled `EpicQuadPanorama` sky
sphere (`AdvancedPreviewScene.cpp:62-86`). This host does not override the rotation —
`Config/DefaultEditor.ini:27-29` pins all three profiles to the same value.

Both halves are fixed, and neither was chosen with reference to the other, so **every** zero-argument
capture of **every** asset lands at the same azimuth relative to both.

## Measured

`Docs/wiki-src/visual-review.md:142` and `Docs/wiki-src/model.authoring.md:42` record, at a pinned
`ev100 -1`: rook `meanLuminance` **0.143** default vs **0.285** opposed, driftwood 0.174 / 0.351,
crane 0.167 / 0.255, mobius 0.148 / 0.255. `model.authoring.md:42` adds the consequence directly:
"That frame is where 'inverted normals' reports come from" — three separate false reports.

## The mechanism in the original report is inverted

The report said "camera on −X, light on +X". Both are on −X. `FRotator(-40, -67.5, 0)` forward is
`(+0.293, −0.708, −0.643)`; photons travel that way, so the **source** sits toward
`(−0.293, +0.708, +0.643)` — azimuth **112.5°**, on the −X/+Y/+Z side.

- Default camera forward `(+1, 0, 0)`: `dot(forward, toLight) = −0.293` — the light is *behind* the
  camera. The default pose is **front-lit**.
- Opposed camera forward `(−1, 0, 0)`: `dot = +0.293` — the light is *in front of* the camera. The
  opposed pose is the **back-lit** one.

So the luminance gap is not "the default camera faces away from the key light". It is a **frame
mean** dominated by the sky-sphere backdrop and the 4×-scaled grid floor: the opposed camera sits
67.5° from the sun's azimuth against the default camera's 112.5°, so it frames the bright half of the
panorama. Higher frame-mean luminance is not evidence the subject is better shaded. The measurement
and the false-diagnosis consequence both stand; only the stated cause is wrong, and it matters
because a fix aimed at "turn the camera toward the light" would be aimed at the wrong quantity.

## The wave did not touch it

The only wave commit touching `RenderHandler.cpp` is `5e66108e`, whose entire diff there adds
`RPC_PARAM_OPT("hideEditorSprites", …)` to a **sibling** verb. `git log -S"CameraLocation = Center +
FVector(-Distance"` last hits `8962f163` (2026-06-23, a rename), so the block predates the wave.

## Documented in four places, none of them the method's own page

`Docs/wiki-src/visual-review.md:11` ("the default camera sits on the preview key light's dark side
and renders a correct closed mesh as a black silhouette" — itself carrying the inverted mechanism),
`:142`, `Docs/wiki-src/model.authoring.md:42`, `Docs/wiki-src/visual-review.model-rig.md:24`.

The gap that survives: `Docs/wiki-src/render.md:128-179` is the
`### render.capture_asset_preview` overlay that generates the page an agent actually lands on. It
documents `closeAfterCapture`, `orthoWidth` and the rest, never says `location` is optional, never
states the default pose, and carries no lighting warning or cross-link. The wire description at
`RenderHandler.cpp:228` is just `"Preview camera location {x, y, z}."`

**Fix:** derive the default pose from the rig instead of from world −X — the handler can read
`UAssetViewerSettings::Get()->Profiles[i].DirectionalLightRotation` and place the camera at a chosen
angle to it, which makes the default frame stable under a changed profile. Whatever the pose, put the
default-camera statement on `render.md`'s own overlay section, and correct the "faces away from the
key light" wording in the four pages that carry it.

## Related

- `B-capture-asset-preview-renders-empty` (IN-REVIEW) — the non-realtime white-frame bug; its `#2`
  note is where the −X bounds-fit pose came from. Origin of the pose, not a ticket about it.
- `B-exposure-pin-black-frame` (OPEN), `B-unlit-level-capture-no-warning` (OPEN) — other dark-frame
  causes, explicitly different mechanisms.
- `B-capture-preview-decoration-not-suppressible` — the same docs-placement gap on the same verb.

## History
- `#1-fixed-pose-fixed-rig-decided-apart` `OPEN` reporter — `render.capture_asset_preview`'s zero-argument camera is `Center + FVector(-Distance, 0, Radius*0.35)` looking toward +X (`RenderHandler.cpp:313-317`), an FOV-fit of the bounding sphere with no reference to scene lighting; the scene is an engine `FAdvancedPreviewScene` (`SStaticMeshEditorViewport.cpp:143`) whose light is fixed at `FRotator(-40,-67.5,0)` (`AssetViewerSettings.h:62`, applied `AdvancedPreviewScene.cpp:104`, unoverridden per `Config/DefaultEditor.ini:27-29`) inside a 2000x `EpicQuadPanorama` sky sphere (`AdvancedPreviewScene.cpp:62-86`). Both are fixed and neither was chosen against the other, so every zero-argument capture of every asset lands at the same azimuth to both. Measured at pinned `ev100 -1` (`Docs/wiki-src/visual-review.md:142`, `Docs/wiki-src/model.authoring.md:42`): rook 0.143 default vs 0.285 opposed, driftwood 0.174/0.351, crane 0.167/0.255, mobius 0.148/0.255, and the docs attribute three false "inverted normals" reports to that frame. The reported mechanism is inverted and is corrected here: the light source sits at azimuth 112.5°, so `dot(defaultForward=+X, toLight) = -0.293` puts it *behind* the default camera (front-lit) and *in front of* the opposed one (back-lit); the gap is a frame mean dominated by which half of the panorama backdrop is in shot, not subject shading — which matters because a fix aimed at "turn the camera toward the light" would target the wrong quantity. The wave did not touch this: the only wave commit in `RenderHandler.cpp` is `5e66108e`, adding `hideEditorSprites` to a sibling verb, and `git log -S` on the pose expression last hits `8962f163` (2026-06-23). Workaround (pass explicit `location`/`rotation`) is documented in four overlay pages but not on the method's own page — `Docs/wiki-src/render.md:128-179` never states that `location` is optional, never states the default pose, and carries no lighting warning; the wire description at `RenderHandler.cpp:228` is bare. Fix: derive the default pose from `UAssetViewerSettings::Get()->Profiles[i].DirectionalLightRotation` rather than world -X, move the statement onto `render.md`, and correct the "faces away from the key light" wording in the four pages carrying it.
