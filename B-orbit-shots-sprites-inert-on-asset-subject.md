---
id: B-orbit-shots-sprites-inert-on-asset-subject
title: "camera.orbit_shots, camera.frame_actor and render.capture_annotated accept hideEditorSprites on an asset subject, where it is inert, while refusing previewScene on the level branch where that is inert"
status: OPEN
severity: Low
category: bug
tags: [render, camera, orbit_shots, capture_asset_preview, animation_shots, capture_animation_preview, hideEditorSprites, previewScene, parity, preview-scene, silent-noop, unknown-params]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
rice: [1, 1, 1, 1]
priority: 8
---

# Three hybrid capture verbs accept an inert `hideEditorSprites` on asset subjects

Three capture verbs shoot either the Level Editor viewport (actor/world subject, `actorName`,
`point`) or an asset editor's preview viewport (staticMesh/skeletalMesh/animation/niagara subject).
On all three, `previewScene` is refused on the level branch, where it would be inert, with
`UNSUPPORTED_ASSET_EDITOR` "rather than accepted and quietly ignored". `hideEditorSprites`, which is
inert on the asset branch, is declared, parsed and forwarded on both branches with no branch note:

| verb | declared | parsed | forwarded / applied |
|---|---|---|---|
| `camera.orbit_shots` | `Source/PinWright/Private/Handlers/Render/CameraFrameHandler.cpp:711` | `:833-834` | `:1173` |
| `camera.frame_actor` | `CameraFrameHandler.cpp:261` | `:429-430` | `:534` |
| `render.capture_annotated` | `AnnotatedCaptureHandler.cpp:202` | `PreviewViewportCaptureUtils.cpp:2010` via `:229` | not cleared for `bAssetSubject` (only `bAutoViewDistanceScale` is, `:546`) |

The plugin's own rule says the flag is inert on a preview scene: no billboard-gated component is
ever registered there (`PreviewViewportCaptureUtils.h:44-70`). The pure-preview verbs act on it:
`render.capture_asset_preview` does not declare it and clears it after parsing
(`RenderHandler.cpp:763-799`), and `render.capture_animation_preview` refuses it by name
(`AnimationPreviewCaptureHandler.cpp:405`). So the same asset and viewport get `UNKNOWN_PARAMS` on
one verb and a silent no-op on three others.

The response does not lie about pixels: `viewport.editorSprites` reports `hideRequested:true,
visible:false, restored:true` with no `hideWarning` (`PreviewViewportCaptureUtils.cpp:3215-3240`,
warning gated at `:3229`). It does present the flag as having done something on a scene that never
had a sprite, and each verb's generated wiki page carries the shared description promising icon
suppression with no asset-subject caveat.

**Fix:** on each of the three verbs, after the subject resolves to an asset kind, refuse
`hideEditorSprites:true` with `UNSUPPORTED_ASSET_EDITOR` (the code `previewScene` already uses on
the opposite branch), and append an "LEVEL SUBJECTS ONLY on this verb" note to the parameter
description, mirroring the `previewScene` note. Not asked: declaring the flag on the pure-preview
verbs.

**Acceptance:** `camera.orbit_shots`, `camera.frame_actor` and `render.capture_annotated` with
`subject:{kind:"staticMesh", path:"/Engine/BasicShapes/Cube.Cube"}, hideEditorSprites:true` each
return `UNSUPPORTED_ASSET_EDITOR` before any capture; the same calls with an actor subject still
capture and report `editorSprites.hideRequested:true`; the three method pages show the branch note.

## Related

- `B-capture-preview-decoration-not-suppressible` — the gizmo and grid on the same asset branch,
  which `hideEditorSprites` does not cover.
- `F-preview-turntable-capture` (IN-REVIEW) — the workflow that surfaced it.
- `B-declared-param-guard-blind-to-helpers` — the inverse shape (a verb reads what it does not
  declare).

## History
- `#1-sprites-accepted-on-preview-branch` `OPEN` reporter — "Filed 2026-09-02 from the Atlantis showcase video. `camera.orbit_shots` declares `hideEditorSprites` unconditionally (`CameraFrameHandler.cpp:705`), parses it unconditionally (`:805`) and forwards it unconditionally (`:1133` -> `PoseListCapture.cpp:34`), even though its asset-subject branch shoots an `FPreviewScene`-backed viewport where the flag's three gating component classes are never registered — the conclusion the plugin itself reaches in three other files and acts on by REFUSING the parameter (`PreviewViewportCaptureUtils.h:47-71`; `RenderHandler.cpp:433-451,460`; `AnimationPreviewCaptureHandler.cpp:385-402`), and which `AnimationShotsHandler.cpp:165-169` states as the qualifying rule ('this verb draws the LIVE Level Editor viewport'). The same verb refuses its OTHER branch-sensitive parameter on the inert branch, one line above, with the words 'rather than accepted and quietly ignored' (`:706`). Verified live with two refused-before-capture probes: `camera.orbit_shots` with an asset subject + `hideEditorSprites:true` reaches TOO_MANY_SHOTS (past the param gate), while `render.capture_asset_preview` with the same flag returns UNKNOWN_PARAMS. Severity Low: the response's `editorSprites` block is unconditional and reports `visible:false` truthfully (`PreviewViewportCaptureUtils.cpp:2746-2776`), so no pixel claim is false — what is wrong is that an inert knob is published as live, on the one verb that spans both viewport kinds."
- `#2-rephrased` `OPEN` developer — Old premise ("`camera.orbit_shots` is the one capture verb that shoots both viewport kinds") is outdated: `camera.frame_actor` (`CameraFrameHandler.cpp:261,429,534`) and `render.capture_annotated` (`AnnotatedCaptureHandler.cpp:202`, parsed through `PreviewViewportCaptureUtils.cpp:2010`, never cleared for asset subjects) take asset subjects and accept the inert flag the same way. Widened to the three verbs, refreshed every line reference at 7230b41d (orbit_shots now `:711`/`:833`/`:1173`, editorSprites block `PreviewViewportCaptureUtils.cpp:3215-3240`), dropped the four-verb parity table and the live `count:25` probe (its TOO_MANY_SHOTS refusal now depends on `maxShots`), added Acceptance. Severity unchanged (Low).
