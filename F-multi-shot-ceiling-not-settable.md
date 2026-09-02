---
id: F-multi-shot-ceiling-not-settable
title: "The 24-shot ceiling is one grandfathered constant shared by four capture verbs, refused hard with no caller override and no published per-shot cost — so a set that needs 25 frames costs a second full rig cycle and nobody can argue the number"
status: OPEN
severity: Medium
category: feature
tags: [render, camera, orbit_shots, capture_asset_preview, animation_shots, capture_animation_preview, shot-cap, GMaxOrbitShots, TOO_MANY_SHOTS, preview-scene, budget, turntable]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# One constant, four verbs, no lever and no derivation

Every multi-shot capture verb in the plugin refuses at the same number, and that number is a
literal nobody has re-argued since the first verb shipped.

    constexpr int32 GMaxOrbitShots = 24;
    -- Source/PinWright/Private/Handlers/Render/CameraShotPlanUtils.h:60

## Where it binds

Four verbs, each refusing before any capture runs, each also passing the same value into the
shared primitive as its per-call ceiling:

| verb | refusal | ceiling forwarded |
|---|---|---|
| `camera.orbit_shots` | `Handlers/Render/CameraFrameHandler.cpp:927-931` | `:1141` |
| `render.capture_asset_preview` | `Handlers/Render/RenderHandler.cpp:786-795` | `:941` |
| `camera.animation_shots` | `Handlers/Render/AnimationShotsHandler.cpp:452-458`, `:598-604` | `:817` |
| `render.capture_animation_preview` | `Handlers/Render/AnimationPreviewCaptureHandler.cpp:682-688` | `:1014` |

It also silently sets two *derived* defaults, so it shapes plans that never come near 24:

    FMath::Clamp(GMaxOrbitShots / FMath::Max(ViewPlan.Num(), 1), 1, 5);
    -- Handlers/Render/AnimationPreviewCaptureHandler.cpp:591

    return FMath::Clamp(PinWrightCameraFrame::GMaxOrbitShots / SafeViews, 1, GDefaultFrameCountCap);
    -- Handlers/Render/AnimationShotsHandler.cpp:104

And on the two verbs with a time axis it binds on the **product**, not on either factor:
`render.capture_asset_preview`'s `times` documents this in its own parameter text — *"the combined
total is bounded by the same 24-shot ceiling as count/views"* (`RenderHandler.cpp:349`) — so five
instants of the six axis-aligned sides is 30 and is refused outright. A caller who reasons "6
cameras, well under 24" is wrong for a reason the number itself does not carry.

## The number has no derivation, only a cost sentence and a grandfather clause

The constant's own comment argues that *some* bound is needed, never that this is the bound:

    // Hard ceiling on shots per multi-shot call. Each shot moves a real editor camera, resizes
    // the viewport and does a full offscreen readback, so an unbounded count would stall the
    // editor; beyond this we reject with a typed TOO_MANY_SHOTS.
    -- CameraShotPlanUtils.h:57-59

The shared primitive it feeds has a *different* ceiling with a *stated* derivation and a
different enforcement policy:

    // Hard ceiling on poses per call. ... Eight is the working set an agent can actually look
    // at in one pass.
    //
    // A longer list is TRUNCATED AND REPORTED, never silently shortened
    constexpr int32 MaxPosesPerCall = 8;
    -- Handlers/Render/PoseListCapture.h:43-52

and says in as many words why the four verbs above carry 24 instead:

    // Effective ceiling for THIS call. Defaults to MaxPosesPerCall; a verb that already
    // shipped a different, higher bound sets its own here so converting it onto this
    // primitive does not silently shorten sets its callers have been taking for months
    // (camera.orbit_shots has allowed 24 since it shipped).
    -- PoseListCapture.h:151-155

That is the whole basis on record: **24 is what one verb happened to ship with, preserved for
compatibility.** 8 is derived from what an agent can review; 24 is derived from nothing. The two
differ by 3× on the same primitive, on the same hardware, for the same per-shot work.

**This is not the refuse-vs-truncate asymmetry**, which *is* argued and is correct as it stands
(`RenderHandler.cpp:778-782`: a truncated instants x cameras set "reads as a complete set of the
wrong thing", so a refusal naming both factors is the right answer). This ticket is about the
number and the absence of a lever, not about how the bound is enforced.

## Nothing lets a caller budget against it, and nothing lets a caller accept the cost

- **No parameter raises it.** None of the four verbs declares a `maxShots` / `budget` /
  `allowLongSet`; the dispatcher's strict unknown-param gate refuses anything of the sort by name.
  Both halves verified live against the running editor on `/Game/Maps/Atlantis`, 2026-09-02:

      camera.orbit_shots {subject:{kind:"staticMesh", path:"/Engine/BasicShapes/Cube.Cube"},
                          count:25, maxShots:240}
      [UNKNOWN_PARAMS] Unknown parameter(s) for 'camera.orbit_shots': [maxShots]. Valid
      parameters: [actorName, objectPath, actorPath, actor_name, point, subject, count, angles,
      elevation, radius, padding, fov, width, height, viewMode, distribution, seed,
      projectionMode, views, exposure, hideEditorSprites, previewScene, inline].

      camera.orbit_shots {subject:{...}, count:25}
      [TOO_MANY_SHOTS] Requested 25 shots exceeds the maximum of 24

  One shot over the line is a hard refusal, and there is no budget knob of any spelling to raise it.
- **The bound is published, the cost is not.** The set report emits `maxPosesPerCall`
  (`Handlers/Render/PoseListCapture.cpp:393`) beside `posesRequested` / `posesCaptured` /
  `posesTruncated` (`:390-392`), so the ceiling in force is never implicit — but there is no
  elapsed-time or per-shot-cost field anywhere in the set report. The constant's stated
  justification is a cost argument, and the response publishes no cost. A caller cannot check the
  premise, and neither can a reviewer proposing a different number.
- **The wiki repeats the number without the reason.** `Saved/PinWright/wiki/camera.orbit_shots.md:102`
  — *"More than 24 planned shots gives `TOO_MANY_SHOTS`."*

## What it cost, measured

A 240-frame turntable of one column, 2026-09-02, on `/Game/Maps/Atlantis`: **10 `camera.orbit_shots`
calls of 24 explicit `angles` each**, because 24 is where the verb stops. The `previewScene` rig
is applied once and restored once **per call** by design — the `previewScene` parameter says so
itself, *"Read once for the whole set ... and restored once after the last one"*
(`CameraFrameHandler.cpp:706`); the apply at `Handlers/Render/PreviewSceneRig.cpp:700-709` and the
restore at `:786-789`. So
one continuous spin cost **ten rig cycles**, plus an out-of-band reorder because ten calls'
outputs interleave on disk.

The workflow cost of that is owned by **`F-preview-turntable-capture`**, which folds this ceiling
in as its blocker and asks for a turntable verb. This ticket is the other half, and it stands
whether or not a turntable verb is ever built: **every** long set on **every** one of the four
verbs pays the same multiplier, and a fifth verb added tomorrow inherits the constant with the
grandfather clause still as its only argument.

## Ask

Either of these closes it; both are cheap next to a tenth rig cycle.

1. **Re-derive the number and say so at the constant**, from a measured per-shot cost on a
   representative scene, the way `MaxPosesPerCall = 8` states its own basis. If 24 survives the
   measurement, the ticket closes on the comment alone.
2. **Make it a caller-settable budget**, one parameter shared by the four verbs (the same
   `PINWRIGHT_*_PARAM_DESC` one-definition pattern the family already uses for `padding`,
   `exposure` and `hideEditorSprites`), defaulting to today's 24 so no existing call shape moves.
   The refusal stays for a plan above the caller's own budget; `maxPosesPerCall` already reports
   what was in force, so the response contract needs nothing new.

Publishing a per-shot elapsed cost in the set report would make either version checkable, and is
the piece that lets the next person argue the number instead of inheriting it.

Not asked for: changing the refuse-vs-truncate policy (argued and correct, `RenderHandler.cpp:778-782`),
and not raising the primitive's own `MaxPosesPerCall = 8`, which has its own stated derivation.

**Regression floor to keep:** `Tests/Render/TestCameraFrameSubjects.cpp:141-153` asserts both
constants and asserts they differ, precisely so a "simplifying" edit that drops the per-verb
assignment fails a test instead of quietly shortening sets. Any change here must keep an
equivalent assertion against whatever the new default is.

## History
- `#1-ceiling-has-no-derivation` `OPEN` reporter — "Filed 2026-09-02 from the Atlantis showcase video. `GMaxOrbitShots = 24` (`CameraShotPlanUtils.h:60`) binds four verbs (`CameraFrameHandler.cpp:927-931,1141`; `RenderHandler.cpp:786-795,941`; `AnimationShotsHandler.cpp:452-458,598-604,817`; `AnimationPreviewCaptureHandler.cpp:682-688,1014`), shapes two derived frame budgets (`AnimationPreviewCaptureHandler.cpp:591`, `AnimationShotsHandler.cpp:104`), and binds on the instants x cameras PRODUCT (`RenderHandler.cpp:349`). Its only recorded basis is a generic cost sentence (`CameraShotPlanUtils.h:57-59`) plus an explicit grandfather clause (`PoseListCapture.h:151-155`, 'camera.orbit_shots has allowed 24 since it shipped'), against a sibling constant that does state its derivation (`PoseListCapture.h:43-52`). No parameter raises it and no per-shot cost is published (`PoseListCapture.cpp:390-393` reports the bound, not the cost). Measured cost 2026-09-02: a 240-frame turntable took 10 calls and 10 preview-scene rig cycles. Workflow cost owned by F-preview-turntable-capture; this ticket owns the bound itself."
