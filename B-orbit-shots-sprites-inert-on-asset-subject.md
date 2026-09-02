---
id: B-orbit-shots-sprites-inert-on-asset-subject
title: "camera.orbit_shots is the one capture verb that shoots both a level viewport and an asset preview, and it applies the two branch-sensitive parameters by opposite rules — previewScene is refused on the branch where it is inert, hideEditorSprites is accepted on the branch where it is inert"
status: OPEN
severity: Low
category: bug
tags: [render, camera, orbit_shots, capture_asset_preview, animation_shots, capture_animation_preview, hideEditorSprites, previewScene, parity, preview-scene, silent-noop, unknown-params]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# Same verb, two viewport kinds, two opposite policies for the same problem

`camera.orbit_shots` is the only capture verb whose subject decides *which kind of viewport* it
shoots: `actorName`, a bare `point`, and a subject of kind `world`/`actor` shoot the Level Editor
viewport; a subject of kind `staticMesh`/`skeletalMesh`/`animation`/`niagara` opens that asset's
editor preview. Two of its parameters only mean anything on one of those branches. It handles them
by contradictory rules, in the same `RPC_PARAMS` block, one line apart.

**`previewScene` — refused on the branch where it is inert, and the description says why:**

    ASSET SUBJECTS ONLY on this verb: orbiting `actorName`, a bare `point`, or a subject of
    kind world or actor shoots the Level Editor viewport, which has no preview scene to rig,
    and is refused with UNSUPPORTED_ASSET_EDITOR rather than accepted and quietly ignored.
    -- Source/PinWright/Private/Handlers/Render/CameraFrameHandler.cpp:706

**`hideEditorSprites` — accepted on the branch where it is inert, with no branch note at all:**

    RPC_PARAM_OPT("hideEditorSprites", "boolean", PINWRIGHT_HIDE_EDITOR_SPRITES_PARAM_DESC
        " Read once for the whole set, so every shot in one call shows the same viewport
          decoration."),
    -- CameraFrameHandler.cpp:705

"rather than accepted and quietly ignored" is the correct policy, written by this codebase, on the
line above the parameter that violates it.

## The plugin has already decided that the flag is inert on a preview, three times

This is not a judgement call this ticket is introducing. The sprite flag's reach was re-verified
against UE 5.8 on 2026-08-21 and the conclusion is recorded in three places, each of which
*refuses* the parameter rather than reading it:

- `Handlers/Render/PreviewViewportCaptureUtils.h:47-71` — the shared rationale above the wire
  description. Exactly three component classes gate draw relevance on
  `EngineShowFlags.BillboardSprites` (`UArrowComponent`, `UBillboardComponent`,
  `UMaterialBillboardComponent`); `FPreviewScene` / `FAdvancedPreviewScene` register none of them
  and never `SpawnActor`, so *"the only thing this parameter could ever hide in a preview is a
  user-toggled wind gizmo — never the light bulbs, audio icons or player start it documents.
  Declaring it there would publish a knob whose stated purpose is unreachable, which is worse than
  not offering it."*
- `Handlers/Render/RenderHandler.cpp:433-451` + `:460` — `render.capture_asset_preview` not only
  omits it from `RPC_PARAMS`, it defensively clears `Request.bHideEditorSprites = false` because
  the shared parser would otherwise read it on a direct (non-wire) invocation.
- `Handlers/Render/AnimationPreviewCaptureHandler.cpp:385-402` — *"`hideEditorSprites` is
  deliberately NOT a parameter here, and is not read either ... the parameter is undeclared and
  the dispatcher's unknown-param gate refuses it BY NAME rather than this file reading it and
  silently doing nothing."*

And the one verb that *does* offer it beside `camera.orbit_shots` states the qualifying condition
precisely, which is the condition `camera.orbit_shots`' asset branch fails:

    // Offered here and NOT on the preview verbs, and the difference is real rather than
    // stylistic: this verb draws the LIVE Level Editor viewport, whose world has actors and
    // therefore billboard sprites over the subject. FPreviewScene registers components with
    // no AActor at all, so the same flag on an asset-preview verb could only ever report that
    // nothing changed (PreviewViewportCaptureUtils.h:26-31).
    -- Handlers/Render/AnimationShotsHandler.cpp:165-169

`camera.animation_shots` is a pure level-viewport verb — it declares `hideEditorSprites`
(`:170-172`) and declares **no** `previewScene` at all. `camera.orbit_shots` is the hybrid, and
nothing in its handler branches the flag on which viewport it ended up with: the parse is
unconditional at `CameraFrameHandler.cpp:805` and the value is forwarded unconditionally at
`:1133` into `FPoseListCaptureRequest::bHideEditorSprites`, from where the primitive copies it
onto every frame (`Handlers/Render/PoseListCapture.cpp:34`).

## Verified live, 2026-09-02, no captures taken

Both probes were refused before any viewport was acquired, so neither rendered anything. Running
editor on `/Game/Maps/Atlantis`:

    camera.orbit_shots {subject:{kind:"staticMesh", path:"/Engine/BasicShapes/Cube.Cube"},
                        count:25, hideEditorSprites:true}
    [TOO_MANY_SHOTS] Requested 25 shots exceeds the maximum of 24

The shot-count refusal is *downstream* of the dispatcher's unknown-param gate
(`CameraFrameHandler.cpp:927-931`, after the `RPC_PARAMS` check), so reaching it proves
`hideEditorSprites` was accepted on an asset subject.

    render.capture_asset_preview {assetPath:"/Engine/BasicShapes/Cube.Cube",
                                  hideEditorSprites:true}
    [UNKNOWN_PARAMS] Unknown parameter(s) for 'render.capture_asset_preview':
    [hideEditorSprites]. Valid parameters: [assetPath, subject, filename, target, width, height,
    location, rotation, projectionMode, fov, orthoWidth, orthoWorldWidth, exposure, rejectBlank,
    allowBlank, measureCoverage, viewMode, previewScene, closeAfterCapture, count, views,
    elevation, distribution, seed, time, times].

Two verbs, the same asset, the same preview viewport, opposite answers to the same parameter.

## What the caller is told afterwards

Not nothing — which is why this is Low and not higher — but not the truth either. The
`viewport.editorSprites` block is unconditional (`PreviewViewportCaptureUtils.cpp:2746-2776`) and
on an asset-subject orbit with `hideEditorSprites:true` it reports
`hideRequested:true, visible:false, billboardSprites:false, restored:true` and **no**
`hideWarning` — because the warning fires only on `bHideEditorSprites Requested &&
bBillboardSpritesApplied` (`:2758`). A caller reads a clean "you asked for suppression, nothing is
visible, it was restored" over a scene that never had a sprite in it. The pixels are right; the
inference a caller draws about the flag being load-bearing for their acceptance shots is not.

## The family has no parity rule, only per-verb decisions

Reading the four asset-capable capture verbs' `RPC_PARAMS` blocks side by side, each shared
parameter is decided independently and no two verbs agree on the whole set:

| parameter | `render.capture_asset_preview` | `render.capture_animation_preview` | `camera.orbit_shots` | `camera.animation_shots` |
|---|---|---|---|---|
| viewport kind | asset preview only | asset preview only | **level OR asset preview** | level only |
| `hideEditorSprites` | no, argued (`RenderHandler.cpp:433-451`) | no, argued (`AnimationPreviewCaptureHandler.cpp:385-402`) | **yes, unargued** (`CameraFrameHandler.cpp:705`) | yes, argued (`AnimationShotsHandler.cpp:165-172`) |
| `previewScene` | yes (`RenderHandler.cpp:341`) | yes (`AnimationPreviewCaptureHandler.cpp:220`) | yes, branch-refused (`CameraFrameHandler.cpp:706`) | absent |
| `measureCoverage` | yes (`RenderHandler.cpp:339`) | no | no | no |
| `filename` | yes (`RenderHandler.cpp:319`) | no | no | no |
| `angles[]` | no | yes (`AnimationPreviewCaptureHandler.cpp:208`) | yes (`CameraFrameHandler.cpp:687`) | yes (`AnimationShotsHandler.cpp:152`) |

Three of those rows already have owners — `measureCoverage` is `B-orbit-shots-no-subject-coverage`,
`filename` is `B-orbit-shots-no-filename-stem`, `angles[]`-versus-`filename` is the table in
`F-preview-turntable-capture`. The `hideEditorSprites` row is the only one where the *argument
already exists in source* and one verb contradicts it, and it is the only one nobody owns.

## Ask

Two lines, no new rendering:

1. On `camera.orbit_shots`, apply `hideEditorSprites` only on the level-viewport branch and
   **refuse** it on an asset subject — the same shape and the same typed code the sibling
   parameter already uses one line above (`UNSUPPORTED_ASSET_EDITOR`, `CameraFrameHandler.cpp:706`),
   or a `sprites.ignoredReason` on the response if a refusal is judged too breaking for a
   parameter callers may already be passing harmlessly. Either is honest; accepting it silently
   is not.
2. Amend the parameter's description on this verb to carry the branch note, the way `previewScene`
   and `exposure` already carry theirs, so the generated wiki page
   (`Saved/PinWright/wiki/camera.orbit_shots.md:28`, which today reproduces the shared description
   verbatim with no branch caveat) stops promising icon suppression on a scene with no icons.

**Not asked for:** declaring `hideEditorSprites` on the two preview verbs. The three-site argument
against that is sound and re-verified against the engine; this ticket agrees with it and asks only
that the fourth verb be held to it.

## Related

- `B-orbit-shots-no-subject-coverage` (Medium, OPEN) and `B-orbit-shots-no-filename-stem`
  (Low, OPEN) — the same family-parity question in the other direction: parameters
  `render.capture_asset_preview` has and `camera.orbit_shots` lacks. This ticket is the direction
  nobody had filed.
- `B-capture-preview-decoration-not-suppressible` (Medium, OPEN) — its history `#2(e)` records this
  asymmetry as a *correction* ("so 'UNKNOWN_PARAMS on preview verbs' is a per-verb fact rather than
  a family rule") but explicitly scopes it out as orthogonal to its own subject, which is the axis
  gizmo and grid. Filed here rather than appended there so the observation has an owner.
- `F-preview-turntable-capture` (Medium, OPEN) — the workflow that surfaced it; its comparison
  table is the source of three rows above.
- `B-declared-param-guard-blind-to-helpers` (High, IN-REVIEW) — the inverse shape (a verb *reads*
  what it does not declare). This one declares what it should not read.

## History
- `#1-sprites-accepted-on-preview-branch` `OPEN` reporter — "Filed 2026-09-02 from the Atlantis showcase video. `camera.orbit_shots` declares `hideEditorSprites` unconditionally (`CameraFrameHandler.cpp:705`), parses it unconditionally (`:805`) and forwards it unconditionally (`:1133` -> `PoseListCapture.cpp:34`), even though its asset-subject branch shoots an `FPreviewScene`-backed viewport where the flag's three gating component classes are never registered — the conclusion the plugin itself reaches in three other files and acts on by REFUSING the parameter (`PreviewViewportCaptureUtils.h:47-71`; `RenderHandler.cpp:433-451,460`; `AnimationPreviewCaptureHandler.cpp:385-402`), and which `AnimationShotsHandler.cpp:165-169` states as the qualifying rule ('this verb draws the LIVE Level Editor viewport'). The same verb refuses its OTHER branch-sensitive parameter on the inert branch, one line above, with the words 'rather than accepted and quietly ignored' (`:706`). Verified live with two refused-before-capture probes: `camera.orbit_shots` with an asset subject + `hideEditorSprites:true` reaches TOO_MANY_SHOTS (past the param gate), while `render.capture_asset_preview` with the same flag returns UNKNOWN_PARAMS. Severity Low: the response's `editorSprites` block is unconditional and reports `visible:false` truthfully (`PreviewViewportCaptureUtils.cpp:2746-2776`), so no pixel claim is false — what is wrong is that an inert knob is published as live, on the one verb that spans both viewport kinds."
