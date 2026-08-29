---
id: B-capture-open-level-pose-params-photograph-stale-grass
title: "render.capture_open_level's location/rotation move the render camera but not the landscape-grass build, so a pose-driven capture photographs grass built around the persistent viewport camera — the identical pose reached via editor.set_camera shows a full grass carpet the capture reported as bare ground"
status: OPEN
severity: High
category: bug
tags: [render, capture_open_level, landscape, grass, foliage, vegetation, stale, silent-wrong-data, verification-evidence, pose, viewport-camera, set_camera]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
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
