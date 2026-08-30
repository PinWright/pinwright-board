---
id: B-capture-open-level-pose-params-photograph-stale-grass
title: "render.capture_open_level's location/rotation move the render camera but not the landscape-grass build, so a pose-driven capture photographs grass built around the persistent viewport camera — the identical pose reached via editor.set_camera shows a full grass carpet the capture reported as bare ground"
status: DONE
severity: High
category: bug
tags: [render, capture_open_level, landscape, grass, foliage, vegetation, stale, silent-wrong-data, verification-evidence, pose, viewport-camera, set_camera]
encounters: 2
lastSeen: 2026-08-30T16:00:00+05:00
---

# The pose parameters move the camera; the grass stays where it was

`render.capture_open_level {location, rotation}` renders the frame from the pose you passed and
the landscape grass in that frame is built around a **different** point — the camera the
persistent Level Editor viewport was sitting at before the call. The frame is valid, non-blank,
settled, and wrong about the one thing the capture was taken to answer.

## Measured, same session, same map, one variable

| call | `meanLuminance` | bytes | grass in frame |
|---|---|---|---|
| `capture_open_level` at pose P (first time) | 0.6443 | 844 KB | **none** |
| `capture_open_level` at pose P (repeat, to rule out a one-off) | 0.6443 | 844 KB | **none** |
| `editor.set_camera` to P, then `capture_open_level` at the same P | **0.4796** | 1558 KB | **full carpet** |

Note the direction: grass darkens the frame, so the *lower* mean is the one with *more* grass.
0.6443 -> 0.4796 is not a subtle shift; it is the difference between bare ground and a carpet, and
the file size nearly doubles. Nothing in either response distinguishes the two: both are
`success: true`, both non-blank, both settled.

## What the pose parameters actually drive (attributed)

They drive the **persistent** viewport client, not a temporary view.
`ApplyCaptureCamera` (`Source/PinWright/Private/Handlers/Render/PreviewViewportCaptureUtils.cpp:900`)
calls `ViewportClient.SetViewLocation(Request.Location)` / `SetViewRotation(Request.Rotation)` at
`:933-936` on the same `FEditorViewportClient` the editor is using, and the capture restores the
entry pose on scope exit at `:1692-1694` (`ON_SCOPE_EXIT` opened at `:1675`). So the requested pose
is on the real client, and it is on it only for the duration of the call.

## What nothing does: settle the grass

Landscape grass is built around **camera locations**, on the **world tick**:

- `ULandscapeSubsystem::Tick` gathers cameras from `World->ViewLocationsRenderedLastFrame`
  (`C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeSubsystem.cpp:728-745`, including the
  engine's own comment *"there is a bug here, which often leaves us with no cameras in the editor --
  try to fall back to previous camera position(s)"* and the `OldCameras` fallback that implements
  it), and calls `Proxy->UpdateGrass(*Cameras, InOutNumComponentsCreated)` at `:891-895`.
- `ULandscapeSubsystem` is a tickable world subsystem. Its `Tick` runs from the engine/world tick.

The capture never runs one. Its settle loop
(`PreviewViewportCaptureUtils.cpp:2143-2177`) pumps `PumpViewport` (`:106-123`), which is
`FSlateApplication::PumpMessages` + `Tick(ESlateTickType::All)` + `SceneViewport->Draw()` +
`FlushRenderingCommands()` — Slate and the renderer, no world tick. Grepped across the whole
plugin, the only `World->Tick(` calls are in `Private/Tests/` (7 sites, all automation tests);
non-test handler code never ticks the editor world and never calls `RegenerateGrass`.

So within the call the grass build is never given the new camera, and by the time the editor idles
and does tick, `:1692-1694` has already put the old camera back — so the next world tick rebuilds
around the *original* pose too. The requested pose never becomes a grass camera at all.

That is also why the `editor.set_camera` route works: it leaves the camera there, the editor ticks
between the two RPCs, `ViewLocationsRenderedLastFrame` picks the new location up, and
`UpdateGrass` builds around it before the capture is taken.

**Honest limit on this attribution.** The tick-vs-pump reasoning above is read off the two sources
cited and matches the measured A/B exactly, but it was not proven by instrumenting a live
`UpdateGrass` call. The half that is certain from the code is the part the ticket title asserts:
the pose is applied to and removed from the persistent client inside one call, and nothing in the
capture path settles or rebuilds grass after the move.

## Why this is worse than a missing feature

It fails in the direction that makes a **working** setup look broken, and it invalidates the
evidence a capture verb exists to produce. Every "is the grass there?" check taken by passing a
pose is potentially photographing grass built somewhere else, so a correct scene is reported bare
and the agent goes off to fix something that was never wrong. It is silent: the response carries
no field that could contradict it, and the pixel evidence looks like a clean, settled frame of
bare ground.

The blast radius is not limited to grass. Anything the engine builds around a *camera* rather than
around the view matrix — landscape grass here, plausibly other camera-driven amortised work — has
the same exposure on this path.

## Fix

Two acceptable shapes, in order of preference:

1. **Settle the grass against the requested pose before readback.** `ULandscapeSubsystem` already
   exposes exactly the entry point needed, and it takes the camera list explicitly:
   `LANDSCAPE_API void RegenerateGrass(bool bInFlushGrass, bool bInForceSync, TOptional<TArrayView<FVector>> InOptionalCameraLocations)`
   (`C:/UE_5.8/Engine/Source/Runtime/Landscape/Public/LandscapeSubsystem.h:114-120`; the body's
   `InOptionalCameraLocations` branch is `LandscapeSubsystem.cpp:632-636`). Passing the capture's
   effective location with `bInForceSync = true` makes the grass agree with the frame without
   depending on a world tick landing inside the call.
2. **If not settled, say so.** Publish a measured field on every level capture that moved the
   camera — the pose the grass was last built around versus the pose rendered — and warn when they
   differ. House style is already this: `viewport.showFlagOverrides`, `warmup.settled` and the
   `viewDistance` block are all measurements rather than promises.

Reporting alone is the weaker fix and should not be the whole of it, because the caller has no way
to act on the warning other than to re-issue the call through `editor.set_camera` — which is the
workaround, not the fix.

**Workaround:** `editor.set_camera` to the pose, then `render.capture_open_level` **with the same
pose passed again** (the capture still needs it, since the verb restores whatever it found). Proven
above: 0.6443 -> 0.4796 at the identical pose.

## Same shape as

- `B-set-camera-no-viewport-redraw` (DONE) — a pose applied to the client that the pixels had not
  caught up with. Same family, different stage: that was the *frame* being stale, this is the
  *scene content* being stale relative to a frame that is otherwise current.
- `B-showflag-cvar-override-contaminates-capture` (DONE) — a capture whose frame is not what every
  honesty field in the response says it is. That one was resolved by measuring and publishing the
  contaminating channel; the same remedy shape applies here as fix option 2.
- `B-ortho-capture-renders-no-landscape-grass` (OPEN) — filed the same session. Also grass absent
  from a capture, also unexplained by culling; **that ticket's mechanism is explicitly not
  attributed**, and this ticket's attribution is NOT offered as its cause.

Not a duplicate of `B-ortho-capture-culls-distant-foliage`: this is a perspective capture at a
close pose with nothing distance-culled out of frame, and the same pose renders the grass
correctly once the camera has been moved persistently.

## Severity

**High**, by impact class: *silent wrong / stale data on a normal path* — the caller trusts a
result that is a lie and builds on it. That is the definition, almost verbatim, and the "builds on
it" half is realised here: the frame is verification evidence for a downstream decision.

**Reach modifier declined.** `render.capture_open_level` runs in almost every session, which by the
rubric would bump this to Critical. Declined: the Critical band is "editor crash, or a write that
corrupts or loses asset data", and this verb writes nothing to any asset and takes nothing down.
Frequency makes it the *first* High to work, not a Critical.

## History
- `#1-pose-photographs-stale-grass` `OPEN` reporter — Measured live against a running editor.
  Identical pose captured twice through `render.capture_open_level` gave `meanLuminance` 0.6443
  both times at 844 KB with no grass in frame; `editor.set_camera` to that same pose followed by
  the same capture gave 0.4796 at 1558 KB with a full grass carpet. Attribution re-derived from
  source: the pose is written to the persistent `FEditorViewportClient`
  (`PreviewViewportCaptureUtils.cpp:900`, `:933-936`) and restored on scope exit (`:1692-1694`),
  while the settle loop (`:2143-2177`) pumps only Slate and the renderer via `PumpViewport`
  (`:106-123`) and never ticks the editor world — the plugin's only `World->Tick(` calls are the 7
  in `Private/Tests/`. Landscape grass is built from `World->ViewLocationsRenderedLastFrame` on the
  world tick (`LandscapeSubsystem.cpp:728-745`, `:891-895`), so the requested pose never becomes a
  grass camera. Not instrumented at `UpdateGrass` itself — see the honest-limit note in the body.
  Proposed fix cites `ULandscapeSubsystem::RegenerateGrass`'s explicit camera-locations parameter
  (`LandscapeSubsystem.h:114-120`, `LandscapeSubsystem.cpp:632-636`).
- `#2-pose-drives-grass-build` `IN-REVIEW` developer — "Took fix option 1 AND option 2, because the
  ticket is right that reporting alone leaves the caller with only the workaround. New
  `Handlers/Render/LandscapeGrassSettle.{h,cpp}` (namespace `PinWrightCaptureGrass`):
  `SettleGrassForCapturePose(World, CameraLocation)` hands the capture's own MEASURED eye
  (`FViewportCaptureOutput::EffectiveLocation`, the same value the response publishes as
  `cameraLocation`) to `ULandscapeSubsystem::RegenerateGrass(bInFlushGrass=false,
  bInForceSync=true, {location})`. Called from `CaptureEditorViewportToPng` after the pose is
  applied and measured and BEFORE the first `PumpViewport`, so the instances exist when the pixels
  are read. `bForceSync` is what makes it complete inside the call —
  `ProcessAsyncGrassInstanceTasks` -> `FAsyncTask::EnsureCompletion` (UE 5.8 LandscapeGrass.cpp:3352
  -> :3361). NO viewport camera is moved for the grass and none is left moved: the location
  travels as data, and grass components are RF_Transient (LandscapeGrass.cpp:3159) so the editor's
  own amortised update rebuilds around the user's camera on the next world tick. Verified the
  engine citations here rather than trusting the ticket's: the tick gather is
  LandscapeSubsystem.cpp:733-739 and the `UpdateGrass` call :900; the camera-locations branch is
  :633-635; `RegenerateGrass` is declared at LandscapeSubsystem.h:120 with the same signature on
  5.3 through 5.8, so no version gate is needed. Honesty half: an unconditional, MEASURED
  `viewport.grass` block on every capture (`measured`, `landscapes`, `builtForPose`, `settled`,
  `cameraLocation`, `components` / `componentsBefore`, `instances`, `pendingComponents` /
  `pendingTasks`, `buildMs`) read off `ALandscapeProxy::FoliageCache.CachedGrassComps` +
  `AsyncFoliageTasks` after the build, plus a `grassWarning` when the pose did not drive the build
  (naming an open transaction when `GUndo` is what stopped it — UpdateGrass no-ops there,
  LandscapeGrass.cpp:2854-2857) or when the build did not finish. `settled: true` with
  `instances: 0` is now the measured statement 'this ground is bare'; `settled: false` is 'not
  built yet' — the two readings the response previously could not tell apart. Regression tests in
  `Private/Tests/Render/TestCaptureLandscapeGrassSettle.cpp`: three pure tests over
  `MakeGrassBuildInfoObject` (bare-ground vs unfinished differ in `settled` and only one warns; a
  pose that never drove the build is warned and names the transaction case separately; an
  unmeasured report publishes no zeros) and one live test that builds a 1x1 landscape in-code
  through `landscape.create`, asserts `SettleGrassForCapturePose` reports `builtForPose` with
  `CameraLocation == pose` and nothing pending, then drives a real `render.capture_open_level` and
  asserts `viewport.grass.cameraLocation` equals the response's top-level `cameraLocation` — the
  pixels' own pose. The capture half emits `PINWRIGHT_ASSERTIONS_SKIPPED` with reason
  `level-viewport-capture-unavailable` on a host with no live Level Editor viewport rather than
  passing silently. NOT done, and stated rather than implied: the ticket's luminance A/B is not
  reproduced as a test — it needs an authored grass material no in-code fixture can build, and
  re-deriving it from mean luminance would make the assertion depend on GPU, lighting and
  exposure. What is asserted instead is the exact routing property plus the honesty contract.
  Wiki: a `grass` row and three paragraphs in `docs/wiki-src/render.md`. Not compiled or run here —
  the orchestrator builds and runs the suite. Cross-ticket note for
  `B-ortho-capture-renders-no-landscape-grass`: that ticket's frame goes through this same
  `CaptureEditorViewportToPng` path, so it now gets the same build and the same block; but its own
  evidence (grass absent at a pose the persistent camera already sat at) is not explained by this
  mechanism alone, and a top-down ortho puts the eye thousands of cm above terrain whose grass
  varieties cull at a default 10000 cm — the new `instances` / `cameraLocation` fields are what
  will separate 'never built for this eye' from 'built and culled' on the next measurement."

- `#3-set-camera-route-is-stale-in-time-not-in-space` `IN-REVIEW` reporter — **Second encounter,
  and it lands on this ticket's WORKAROUND rather than on its mechanism. Status deliberately
  unchanged: I was not asked to verify `#2` and am not acting as tester.** Measured during a bulk
  instance re-seat on `/Game/Maps/PW_VegetationTest` (host `EAContentExamples58`, UE 5.8; method in
  `Docs/map/tree-seating-on-slopes.md` § *Captures*). **Scope statement first, because it changes
  how this entry should be read: this checkout is at PinWright HEAD `0f4f9594`, and `#2`'s fix is
  NOT in it** — `Handlers/Render/LandscapeGrassSettle.{h,cpp}` does not exist, `RegenerateGrass` has
  zero occurrences in plugin code, and no `viewport.grass` block is emitted anywhere (verified by
  `git ls-files` and grep over the whole plugin at that HEAD). Several hosts share this board, so
  `#2` was presumably landed in a sibling clone. **Nothing below is a regression report against
  `#2`; it is evidence measured on a build that does not have it.**
  **The finding: `editor.set_camera` — the workaround this ticket publishes — is also stale, in
  time rather than in space, and `warmup.settled` cannot see it.** The pose is left on the
  persistent client and the world does tick, so `UpdateGrass` does get the new camera; it then
  rebuilds at **one grass component per world tick**. `grass.MaxCreatePerFrame` defaults to `1`
  (`C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeGrass.cpp:181-185`), and that is the
  live throttle on the tick path: `ALandscapeProxy::UpdateGrass` (`:2848`) `continue`s out of its
  innermost sub-section loop at `:3121-3124` once `InOutNumCompsCreated >= GrassMaxCreatePerFrame`
  (counter incremented `:3140`), and `ULandscapeSubsystem::Tick` (`LandscapeSubsystem.cpp:686`)
  zeroes the counter every tick and passes `bForceSync = false` (`:897-901`). Measured on this map:
  **60-150 s to repopulate a large cull radius after the camera teleports.**
  **What the capture reports during that window.** `warmup.settled: true, settleRounds: 1` on a
  frame with no grass in it at all, and the same on a repeat 12 s later, still empty. That is not a
  bug in the settle loop — it is the loop answering the question it was written to answer. The
  criterion is frame **mean-luminance** stability between two consecutive redraws
  (`PreviewViewportCaptureUtils.cpp:2158-2169`, tolerances `PreviewViewportCaptureUtils.h:310-311`),
  and `PumpViewport` (`:106-124`) is `PumpMessages` + `Slate Tick` + `SceneViewport->Draw()` +
  `FlushRenderingCommands()` with **no world tick anywhere** — so none of its rounds can advance the
  subsystem the grass build rides on. Two consequences worth stating separately:
  **(a) The field is a frame measurement being read as a scene-readiness signal**, exactly as its
  own comment says it is (`:3066-3068`, *"a measurement rather than a promise"*). **(b) The signal
  is inverted at the extreme**: an unbuilt frame is a *stable* frame, so it settles on the first
  round after the three unconditional pre-pumps (`:2090-2093`) — `settleRounds: 1`, the most
  confident-looking value the field can take, is what the emptiest scene produces, while a scene
  that is actively building would report more rounds. And the obvious defence fails with it: a
  repeat-until-stable check confirms the same frame, because the two shots agree *because* the build
  has not progressed.
  **Measured ladder at one pose** (recorded in `Docs/wiki-src/render.md:120` at `22461cdd`):
  `meanLuminance` 0.305 with no grass, 0.278 after 45 s, 0.275 after 105 s, against 0.251 for the
  fully built carpet the same pose had shown earlier. The cost to a before/after pair is that the
  missing grass moves the frame **further than the edit under test does** — this pass's after-frames
  carry visibly thinner grass than its before-frames, which is the async rebuild and not a change
  anyone made.
  **Documentation half already landed, in the plugin repo, not here:** `22461cdd`
  ("Warn that warmup.settled does not wait for view-driven geometry") added
  `Docs/wiki-src/render.md:120` and `:122` — docs only, +4 lines, no code. It **retires the
  "`set_camera`, then one cheap RPC, then capture" recipe**, replacing it with: move the camera,
  *wait*, then capture repeatedly at the fixed pose until two consecutive frames agree on
  `imageStats.meanLuminance` **and** `sizeBytes`, and compare a pair only when both members are
  built. `sizeBytes` is the more sensitive of the two, since a sparse carpet compresses well. This
  ticket's **Workaround** line should be read together with that page: `editor.set_camera` plus the
  same pose is necessary and is not sufficient.
  **What this means for `#2`, offered for whoever tests it.** `#2`'s route should cover this and not
  merely the pose half, because `ULandscapeSubsystem::RegenerateGrass`
  (`LandscapeSubsystem.cpp:613`, forwarding to `UpdateGrass` at `:669`) with `bInForceSync = true`
  makes `!bForceSync` false at `LandscapeGrass.cpp:3121` and **bypasses the per-frame cap entirely**
  — the engine does this itself at `LandscapeGrass.cpp:1020`. But the throttle is the half a test can
  accidentally not exercise: a fixture on a 1x1 landscape builds its grass in a handful of ticks
  whether or not the fix is present. Suggested addition to `#2`'s live test, stated as a suggestion
  and not a verdict: capture **immediately** after a camera move to a fresh, far pose with no wait
  and no intervening RPC, and assert `viewport.grass.instances` is non-zero and `settled` true on
  that **first** shot. Without that, the test cannot distinguish "force-synced" from "the throttle
  never bit".
  **No new ticket filed, deliberately.** The general form — `warmup.settled` names the frame while
  callers read it as the scene — is already claimed by this ticket's own § *Why this is worse than a
  missing feature* (*"Anything the engine builds around a camera rather than around the view matrix
  ... has the same exposure on this path"*), and the measured readiness block a separate ticket would
  ask for is exactly `#2`'s honesty half. Filing it again would fragment one fix across two tickets.
  **No severity change proposed:** High already, same impact class (silent stale data on a normal
  path, consumed as verification evidence), and a second observation is an `encounters` input, never
  a severity input. `encounters` 1 -> 2.
- `#4-pose-drives-the-grass-build-verified` `DONE` verifier — 2026-08-30, **13:32 build**, editor pid 18592, `/Game/Maps/PW_VegetationTest`. `#2`'s `Handlers/Render/LandscapeGrassSettle.{h,cpp}` and the `viewport.grass` block are present in this checkout and live — `#2` was written against a tree that did not yet have them, so this is the first time they have been exercised here.

  **The deciding evidence is a fact about the viewport camera, which no field of the capture response supplies.** Three `render.capture_open_level` calls at three poses — `(0,0,0)`, `(18000,18000,2000)`, `(-18000,-18000,2000)` — each returned `grass.builtForPose: true` with `grass.cameraLocation` equal to **the requested pose**, and `componentsBefore` → `components` of 136→190, 149→155 and 149→184 with `buildMs` 50.9 / 18.2 / 32.8. Then, read through `python.execute` against `UnrealEditorSubsystem::get_level_viewport_camera_info` — outside the capture verb entirely — **the persistent viewport camera was at `(21000, 5800, 1600)`, pitch -3, yaw 0, and had not moved.** That is exactly `#1`'s defect inverted: grass was built at three locations the viewport camera never occupied, which under `#1`'s mechanism (the build anchored to the persistent camera) is impossible.

  **Confirmed on pixels, not only on numbers**, per this project's standing rule. `pw_grass_poseC_posed.png` at `(-18000,-18000,2000)` with exposure pinned (EV100 -0.5, `adaptedSource: fixedPin`, so auto-exposure cannot be cancelling the change) was looked at: a dense grass carpet across the whole meadow, wildflowers through it, rocks seated in it, trees on the ridge. `#1`'s failing comparison was "the identical pose reached via `editor.set_camera` shows a full grass carpet the capture reported as bare ground" — the pose-driven capture now *is* the full grass carpet.

  **`#3`'s 60–150 s regrowth window is closed as well, and by mechanism rather than by patience.** `#3` measured the `set_camera` workaround as stale in time because `grass.MaxCreatePerFrame` is 1 and repopulating costs hundreds of frames. `#2`'s settle does not wait for frames: `SettleGrassForCapturePose` drives `ULandscapeSubsystem::RegenerateGrass` with `bForceSync` and the capture's own eye position (`LandscapeGrassSettle.cpp:57`), which is why the three builds above cost 18–51 **milliseconds**. Worth recording that `grass.MaxCreatePerFrame` appears nowhere in plugin source — it is named only in `Docs/wiki-src/render.md:127` and `Docs/wiki-src/vegetation-authoring.md:273` — so nothing reads or reports it; the cvar is bypassed, not tuned. `warmup.settled` reported `true` throughout, as `#3` warned it would, and it is **not** what this close rests on.

  **One field of the new block is defective, and it is filed rather than folded in.** `grass.instances` read **0** on all three captures while `components` moved and the frame was visibly full of grass. Confirmed independently: `Landscape_0` holds 149 grass HISMs and `get_instance_count()` sums to 0 across all of them, because `LandscapeGrassSettle.cpp:40-50` sums `GetInstanceCount()` = `PerInstanceSMData.Num()`, an array landscape grass never populates. New ticket **`B-capture-grass-instances-always-zero`** (High) owns it. This ticket is closed anyway because its subject is *whether the pose drives the build*, which is verified three ways above; a wrong density number in the reporting `#2` added is a different defect, and reopening here would put two tickets on one fact.

  **Not checked:** no capture emitted a `reach` block (`bReachMeasured` false), so "built but culled out of frame" — one of the three things `#2`'s `MeasureGrassFrameReach` exists to separate — was never exercised, and the three `grassWarning` texts at `LandscapeGrassSettle.cpp:277`/`:296`/`:322` were never triggered. Also unexercised: the ortho path, which emits the same block from `OrthoTileCaptureHandler.cpp:832`/`:952` and is separately tracked by `B-ortho-capture-renders-no-landscape-grass`.
