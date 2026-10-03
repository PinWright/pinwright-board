---
id: F-multi-shot-ceiling-not-settable
title: "The 24-shot ceiling is one grandfathered constant shared by four capture verbs, refused hard with no caller override and no published per-shot cost — so a set that needs 25 frames costs a second full rig cycle and nobody can argue the number"
status: DONE
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

**The one measurement on record proves 24 is safe, never that 25 is not.** The board's own
`F-animated-capture-verbs` (DONE) carries the only empirical note behind the number:

    - **24 shots = 24 viewport resize cycles is safe.** Two back-to-back 24-shot bursts (48
      cycles, on top of ~48 earlier in the same session) against the `FViewport::GetHitProxy`
      assert class that cost 66 creeps: editor alive and responding, **0** log matches for
      `GetHitProxy|Assertion failed|Fatal error`.
    -- F-animated-capture-verbs.md:60-63

That is a *lower* bound — the longest burst anybody validated — recorded as a clearance for the
value already in the code, not as a ceiling anybody found. It is exactly the shape of evidence
that cannot justify a refusal at 25.

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
- `#2-only-evidence-is-a-lower-bound` `OPEN` reporter — "Additional evidence, same day. The sole empirical note behind 24 is `F-animated-capture-verbs.md:60-63`: two back-to-back 24-shot bursts (48 resize cycles) ran clean against the `FViewport::GetHitProxy` assert class. That clears 24; it establishes no upper bound, so it cannot be the basis for refusing 25. Recorded here because a re-derivation (Ask 1) starts from that measurement."
- `#3-budget-settable-and-cost-published` `IN-REVIEW` developer — Both asks done. **Ask 2:** one parameter, `maxShots` (integer, default 24 so no call shape moves, at most 360), declared on all four verbs through one shared description macro `PINWRIGHT_MAX_SHOTS_PARAM_DESC` and parsed by one helper `PinWrightCameraFrame::ParseMaxShots` (`CameraShotPlanUtils.h`): non-integer / < 1 is `INVALID_ARGUMENT`, above 360 is `TOO_MANY_SHOTS`. Each verb refuses its plan (the instants x cameras PRODUCT on the time-axis verbs) against the caller's budget and forwards the same value as `PoseRequest.MaxPoses`; refusal texts now name `maxShots`. The refuse-vs-truncate policy is unchanged, and the two derived frame-count defaults stay on the default 24. **Ask 1:** the constant's comment now states its actual basis — 24 is grandfathered as the default; the viewport-resize assert class is driven by size VARYING, which `FPoseListCaptureRequest` already makes set-level, so set length adds no instance of it; and the one measured cost on record (24 shots at 256 px = 4.84 s wall, ~0.2 s/shot, `PinWright.camera.orbit_shots.OrbitSetStillAllowsTwentyFour` on Linux Vulkan offscreen). The hard ceiling `GMaxShotsCeiling = 360` is derived per rpc-design §9 from the 120 s response timeout (360 x 0.2 s ~ 72 s). **Published cost:** `RunPoseListCapture` times itself; every `poseSet` carries `elapsedMs` and `msPerShot`, so the number can be re-measured per scene instead of inherited. Regression floor kept: `PoseBoundIsTheVerbsOwnNotThePrimitives` still asserts 24 vs 8 and still passes unmodified. Files: `Handlers/Render/CameraShotPlanUtils.h`, `CameraFrameHandler.cpp`, `RenderHandler.cpp` (capture_asset_preview), `AnimationShotsHandler.cpp`, `AnimationPreviewCaptureHandler.cpp`, `PoseListCapture.h/.cpp`; parity-test note for `MaxPoses`; docs `camera.md`, `render.md`, `render.capture-time.md`, `visual-review.md`; CHANGELOG. Tests (`Tests/Render/TestMultiShotBudget.cpp`): `PinWright.render.shot_budget.ParseMaxShotsBounds`, `.EveryMultiShotVerbReadsMaxShots` (all four verbs declare it as an optional integer, and through the real dispatcher refuse `maxShots:0` as INVALID_ARGUMENT and 361 as TOO_MANY_SHOTS), `.BudgetMovesTheRefusal` (orbit: 25 shots refused by default, accepted with `maxShots:25` — proven by the next refusal being ACTOR_NOT_FOUND — and 4 shots refused under `maxShots:3`; capture_asset_preview: 6 shots refused under `maxShots:5`), `.PoseSetPublishesPerShotCost` (stub capturer; `msPerShot == elapsedMs / 3`), and the live `PinWright.camera.orbit_shots.LongSetOverTheOldCeilingCarriesCoverage` (25 shots captured, `maxPosesPerCall` 25). Not done: `render.capture_mesh` (newer, own renderer, its own `count` 1-24 check) still uses the 24 directly — it is not one of the four verbs and does not build the pose-list primitive.
- `#4-review-fixes` `IN-REVIEW` developer — Review found the 360 ceiling bounded shot COUNT but not cost (rpc-design §9): at 256 px with coverage on by default, 360 shots predict ~149 s, past the 120 s response timeout, and size goes to 16384. Added a time bound on the cost driver: `PinWrightCameraFrame::CheckShotSetCost` / `PredictShotSetSeconds` (`Handlers/Render/CameraShotPlanUtils.h`) predict `shots x draws x (0.2 s + 0.112 s/MP)` (0.2 s/draw measured on the 24-shot orbit; 0.112 s/MP is the ortho-tile readback+encode worst case; coverage = two draws per shot) and refuse with `TOO_MANY_SHOTS` above 90 s, naming the predicted seconds and the levers. Applied in all four verbs once plan and size are known, before anything opens: `CameraFrameHandler.cpp` (coverage counted only with a `subject`), `RenderHandler.cpp` (`capture_asset_preview`), `AnimationShotsHandler.cpp`, `AnimationPreviewCaptureHandler.cpp` (no coverage). The 360 comment and the `ParseMaxShots` refusal no longer claim 360 is derived from the timeout; it is the count bound only. **Behaviour change:** large sizes or coverage now shorten the longest accepted set (e.g. 24 shots at 4096 px with coverage, 360 subject shots at 256 px with coverage). Docs: `camera.md` maxShots bullet (also notes `msPerShot` excludes editor open and rig apply), `render.md` set-budget sentence, CHANGELOG. Test: `PinWright.render.shot_budget.PredictedTimeRefusesPastTheTimeout` (pure model + orbit/asset-preview refusals; an actor-target 360-shot orbit at 256 px is accepted and fails on ACTOR_NOT_FOUND).
- `#5-review-fixes-2` `IN-REVIEW` developer — The re-review NIT is fixed: the cost model now counts the set's full-size warm-up draw (`ShotSetDraws = shots x (coverage ? 2 : 1) + 1` in `CameraShotPlanUtils.h`), both in the prediction and in the refusal message. Before this, 3 shots at 16384 px predicted 86.7 s and were accepted; they now predict ~115 s and are refused. `camera.md` now says to keep `msPerShot` x frames under 90 s and not to double it for coverage. `showcase-video.md` now describes the shipped one-call long-set behaviour. Test `PinWright.render.shot_budget.PredictedTimeRefusesPastTheTimeout` now asserts that the warm-up and per-shot coverage draws are counted.
- `#6-verified-linux` `DONE` tester — Verified on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d). run3/full, all non-skipped: `PinWright.render.shot_budget.ParseMaxShotsBounds`, `.EveryMultiShotVerbReadsMaxShots` (all four verbs declare `maxShots`; through the dispatcher 0 gives INVALID_ARGUMENT and 361 gives TOO_MANY_SHOTS), `.BudgetMovesTheRefusal` (25 orbit shots are refused by default and accepted with `maxShots:25`; capture_asset_preview obeys a lower budget), `.PoseSetPublishesPerShotCost` (`elapsedMs`/`msPerShot`), `.PredictedTimeRefusesPastTheTimeout` (the cost model counts coverage and warm-up draws), `PinWright.camera.orbit_shots.LongSetOverTheOldCeilingCarriesCoverage` (25 shots captured live, `maxPosesPerCall` 25), `.OrbitSetStillAllowsTwentyFour`, and the regression floor `.PoseBoundIsTheVerbsOwnNotThePrimitives`, which passes unmodified. Ask 1 (the constant comment states its measured basis, 4.84 s for 24 shots) and ask 2 (one shared caller budget, default 24) are both met, and per-shot cost is published. Behaviour change in CHANGELOG: a set whose predicted time exceeds 90 s is refused. Not covered: `render.capture_mesh` keeps its own 1-24 count; it is not one of the four verbs.
