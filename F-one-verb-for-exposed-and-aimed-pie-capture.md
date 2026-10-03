---
id: F-one-verb-for-exposed-and-aimed-pie-capture
title: "No single verb can take a correctly exposed PIE frame containing a named actor — screenshot_window aims but cannot expose, editor.screenshot exposes but cannot aim"
status: DONE
severity: Medium
category: feature
tags: [capture, screenshot, exposure, pie, viewport]
---

# No single verb can take a correctly exposed PIE frame containing a named actor

Two capture verbs each solve half of the same job, and the halves do not compose.

| | aims at a chosen pose | exposure control | editor chrome |
| --- | --- | --- | --- |
| `editor.screenshot_window` | **yes** — follows `editor.set_camera` | **no** parameter | yes, whole window |
| `editor.screenshot` | **no** — `captureSource: gameViewport`, ignores `set_camera` | **yes** — `exposure {mode:"fixed", ev100:N}`, verified `pinned:true` | no, clean frame |

So "a correctly exposed frame containing this actor" is currently not expressible in one call.

## How this actually plays out

Photographing an AI firing a rifle, over four consecutive builds of this project:

- Through `screenshot_window`: the actor is in frame, and the muzzle flash — a strong local light —
  blows the shot out, because the only exposure lever available is `ShowFlag.EyeAdaptation 0`, which
  pins a fixed EV with no way to choose it. Leaving auto-exposure on instead makes successive frames
  incomparable, which the `editor.screenshot` docs themselves warn about.
- Through `editor.screenshot {exposure:{mode:"fixed", ev100:1}}`: exposure is correct and repeatable
  and the frame is clean, and the actor is not in it — twice, even with the world frozen at
  `set_global_time_dilation(0.0001)` so the subject could not move between framing and shutter, and
  with the camera pose computed from the pawn's own transform.

The workaround is to accept one defect per frame: a blown-out frame that contains the subject, or a
well-exposed frame of empty scenery. Neither is usable evidence.

## Ask

Either of these closes it; the first is smaller:

1. **`exposure` on `editor.screenshot_window`** — the same `{mode:"fixed", ev100:N} | {mode:"auto"}`
   spelling already implemented for `editor.screenshot`, applied to the level-editor viewport for the
   duration of the grab and restored after. The rendering path already supports a scoped exposure
   override; this is plumbing an existing parameter to a second verb.
2. **Aiming for `editor.screenshot`** — a `location`/`rotation` pair, or a `viewTarget` actor name,
   applied to the game viewport for the shot. `editor.console_command` already accepts a
   `world:'game'` selector, so a `world` selector on `editor.set_camera` would achieve the same and
   be useful on its own.

A `cropToViewport` option on `screenshot_window` would be a welcome extra — the editor chrome is
dead weight in every frame — but it is not what blocks the job.

## Notes

- Adjacent to `B-no-way-to-capture-pie-pixels` `#7` (commit `3dce460`), which records the same split
  as the residue of that bug. Filed separately as a feature because the bug is "PIE pixels are
  uncapturable" (largely fixed — both routes now return real frames) whereas this is "the two routes
  have disjoint capabilities", which needs a design decision rather than a fix.
- Also adjacent to `F-pie-capture-fixed-size-with-umg` and `F-editor-viewport-screenshot`, neither of
  which mentions exposure.
- Workaround available to a project, and what this stream is doing next: put a `PostProcessVolume`
  in the level with manual exposure so the scene's own EV is sane, then use `screenshot_window`. That
  fixes it per-level by changing the content, which is not a substitute for a capture-time control.

## History

- `#1-screenshot-location-rotation` `IN-REVIEW` developer — Took ask 2: `editor.screenshot` gains optional `location` / `rotation`. On the game/PIE branch the existing draw-scoped extension overrides `ISceneViewExtension::SetupViewPoint` (called by `ULocalPlayer::GetViewPoint` after the camera manager, before the view and its culling are built), so the pawn, controller, control rotation and camera manager never move; the first aimed view sets `bCameraCut` so TAA/TSR/motion-blur history from the live pose does not smear in. The pose the view was actually built from is read back in `SetupView` and published as `aim {applied, location, rotation}`; a miss fails `CAPTURE_CAMERA_NOT_APPLIED` and writes no file. On the Level Editor fallback the halves replace the effective pose in the capture request (restored by the shared capture). Combined with `exposure` this is one call for an exposed, aimed PIE frame. Files: `Utils/ScreenshotUtils.{h,cpp}`, `Handlers/Editor/ViewportHandler.cpp`, `docs/wiki-src/editor.md`, CHANGELOG. Test: `PinWright.editor.screenshot.PieExposureAndAim` (owned host-neutral PIE; fixed-size auto, then fixed+aimed captures) and `PinWright.render.capture_open_level.PieWorldWarning` (pure text rule + source contract on the handler wiring), both in `Source/PinWright/Private/Tests/EditorOps/TestPieCaptureExposureAim.cpp` — asserts the reported view pose equals the request, the pin governs it, and afterwards the number of view extensions active on the game viewport is back to its pre-capture count, i.e. the aim extension (which `ULocalPlayer::GetViewPoint` would otherwise keep applying) did not leak. Limits (documented in `editor.md`): the shader-readiness drain is sampled at the live pose before the aimed draw, so the aimed frame gets no warm-up; auto+aim exposure readback is the live pose's. The extension is now released via `ON_SCOPE_EXIT` before the viewport-size restore on every path, including early errors. Non-object `location`/`rotation` are refused `INVALID_ARGUMENT`; fails on revert (no `aim` block). Not done: `exposure` on `editor.screenshot_window` (ask 1) and `cropToViewport`. Syntax-checked with the clang -fsyntax-only fastcheck (UBT module flags), all four .cpp OK; no UHT-visible declarations changed; not built or run yet.
- `#2-verified-linux` `DONE` tester — Verified on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d). run3/full: `PinWright.editor.screenshot.PieExposureAndAim` passed with no skip marker. On an owned PIE session, one `editor.screenshot` call with `exposure:-1.5` plus `location`/`rotation` came back with an `aim` block. `applied` was true, and the reported view pose equals the request (0.5 uu, 0.05 deg). The pin governed the same view (`pinned`, `fixed`, `ev100` -1.5, `adaptedSource:"fixedPin"`). The active view-extension count on the game viewport returned to its pre-capture value, so the aim does not leak into live frames. This meets ask 2, and the ticket says either ask closes it. `docs/wiki-src/editor.md` was checked for the location/rotation section and its limits. Not done, and not required by the ticket: ask 1 (`exposure` on screenshot_window) and the optional `cropToViewport`. Documented limit: the aimed frame gets no shader warm-up at the aimed pose.
