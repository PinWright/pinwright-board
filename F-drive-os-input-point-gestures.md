---
id: F-drive-os-input-point-gestures
title: "No os_input pointer gesture at an arbitrary viewport point or world actor, and no hold/move-while-pressed control, so real-mouse tests of 3D viewport interaction fall back to raw xdotool with no safety gates"
status: IN-REVIEW
severity: Medium
category: feature
tags: [drive, os-input, xtest, drive.click, drive.drag, pie, game-viewport, world-to-screen, gestures]
encounters: 1
costly: 1
lastSeen: 2026-09-29T13:00:00Z
---

# Real-mouse gestures on 3D content need raw xdotool

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv`, plugin `8fcc0b2a`. The task was to reproduce a
map-editor bug where clicking placed objects does not select them, using faithful OS input. That meant:

- a click on a 3D actor in the PIE game viewport (a gate banner, a gizmo arrow);
- a gizmo drag and a box-select drag on the viewport;
- a click whose pointer moves 4-12 px while the button is held (an unsteady hand), a zero-hold click,
  and a right-button drag.

`drive.click` only accepts a `handle` from `drive.observe`, so it cannot target a viewport pixel or an
actor. It also has no hold-duration or move-while-pressed parameters. `drive.drag` has no `os_input`. So
I scripted `xdotool mousemove / mousedown / mouseup`. I mapped screenshot pixels to desktop pixels by
hand, by matching a UMG element's `geometry.absolute` against the screenshot, and I added my own
check that the X window under the pointer was my editor. Batching roughly 150 trials through a direct
HTTP helper (for `object.call_function GetSelection` readback) was the other half of the fallback.

**Ask:**
- `drive.click` / `drive.drag` accept a target point instead of a handle: game-viewport pixel
  coordinates, or `{actor: <path>, component?}` projected through the PIE player's view.
- `os_input` gains `button`, `hold_ms` and an optional `path` of intermediate points while pressed.
- `drive.drag` gains `os_input`.
- The os_input gates from `B-drive-click-os-input-foreign-x-window` and
  `B-drive-os-input-own-window-occlusion` apply to these gestures too.

## History
- `#1-map-editor-selection-repro` `OPEN` reporter - Filed from the PDS map-editor first-click selection repro (board `B-mapeditor-first-click-select` in the game repo). The whole real-mouse matrix ran on raw xdotool. Because xdotool has no occlusion gate, part of one batch went into a peer session's fullscreen game (see `B-drive-click-os-input-foreign-x-window` #3). Costly: about 1 h of scripting plus one lost batch.
- `#2-os-gesture-verb-and-drag-os-input` `IN-REVIEW` developer — New verb **`drive.os_gesture`** rather than making `handle` optional on `drive.click`/`drive.drag`. Those verbs are built on `RunAction`: it re-resolves a UMG handle, diffs UMG elements and injects in one synchronous call, so the engine pumps a whole os_input click in **one frame**. Game code that polls the mouse per tick (a gizmo drag, a box select) never sees the button held that way. `drive.os_gesture` takes `at` = `{x,y}` PIE game-viewport pixels or `{actor, component?}`. Actor targets aim at the bounds center and are projected through the local PlayerController (`OUT_OF_BOUNDS` behind the camera or outside the viewport). It also takes `to`, a pressed `path` of `{dx,dy}` offsets, a strict `button` (left/right/middle), `hold_ms` 0-10000 and `world`. The gesture is a timed step plan (`FDriveOsGesture::BuildPlan`) that a core ticker runs, sending every step due at a tick in that tick. So press, pressed motion and release land on different frames, and `hold_ms:0` with no motion presses and releases in one frame (the zero-hold click). Gates, all before any motion: Slate window order at the press point and the foreign X window (`TARGET_OCCLUDED`), the per-display lock held for the whole gesture (`OS_INPUT_BUSY`) and pointer grab (`POINTER_GRABBED`). Right before the press, `CheckPress` runs (`INPUT_FAILED` for a raised foreign window, `POINTER_MOVED`), and the button stays up if it refuses. Only the press is gated, because after a press X routes motion and release to the pressed window. The response carries the press/release viewport and screen points, the resolved `target`, `hit_actor` (a visibility trace at the press pixel, read before anything moves), `press_frame`/`release_frame` (`GFrameCounter`), `pointer_after` and `pointer_on_release`. **`drive.drag` now takes `os_input`**: left button, 80 ms hold, motion to the release point over `duration_ms`, run blocking through the same plan (`FDriveOsGesture::RunBlocking`), with RunAction's os_input gates. The web surface refuses it like click/hover. `FDriveOsInput` gained the shared primitives `BeginGesture` (lock + grab refusal + pointer read), `CheckPress`, `SendMotion` and `SendButton`. ClickAt's pre-press checks and MoveUnlocked's grab refusal were extracted into them, so there is no copy. Files: `Handlers/Drive/DriveOsGesture.{h,cpp}`, `Handlers/Drive/DriveOsGestureHandler.cpp` (new); `DriveOsInput.{h,cpp}`, `DriveActionHandlers.cpp` (drag os_input + macro text), `DriveWebHandlers.cpp` (DragWeb refuses os_input); `Tests/Drive/TestDriveOsGesture.cpp` (new); `docs/wiki-src/drive.md` (drag os_input + `### drive.os_gesture`); `CHANGELOG.md`. Tests (filter `PinWright.drive.os_gesture`): `.Plan.ApproachPressHoldRelease`, `.Plan.PressedPathVisitsEveryPoint`, `.TickPacing.HeldButtonSpansFrames` (fails if the plan is run in one frame), `.TickPacing.RefusedPressStopsTheGesture`, `.ViewportPixelToScreen`, `.ArgsRefusedBeforeInjecting`, `.RefusedWithoutPie`, `.ParamsDeclared` (also asserts `drive.drag` declares `os_input`). Linux only: `.WaitsForDisplayLock` (~5 s, no X needed) and `.PointerGrabRefusesBeforeMotion` (needs DISPLAY, otherwise skips with the marker). **Not covered by an automated test:** an end-to-end PIE gesture on a real actor. That needs a visible editor on the shared display, and it moves and presses the real pointer.
- `#3-fixround-segment-timing-and-waypoints-rename` `IN-REVIEW` developer — The suite run raised two failures. (1) `PinWright.drive.os_gesture.Plan.PressedPathVisitsEveryPoint` got 100 where it expected 60: a short pressed segment, such as a 4 px wobble, began moving 40 ms after the hold ended. The cause was that `AddSegment` timed each distinct point by its index in the 24-step interpolation, and the first five steps rounded onto the starting pixel. That contradicted the documented `hold_ms` contract ("held before the pointer moves on"). `AddSegment` now spreads only the distinct points evenly over the segment duration, with the first one exactly at the end of the hold. The assertion was correct; the code was wrong. (2) `PinWright.infra.contract.PathParamTypes.PathShapedParamsDeclareAPathType` flags any parameter whose name ends in `path`. The pressed-offset list was renamed from `path` to `waypoints` across the handler, tests, `drive.md` and `CHANGELOG.md`; it is not an asset path, so it is not allow-listed. Files: `Handlers/Drive/DriveOsGesture.cpp`, `Handlers/Drive/DriveOsGestureHandler.cpp`, `Tests/Drive/TestDriveOsGesture.cpp`, `docs/wiki-src/drive.md`, `CHANGELOG.md`.
