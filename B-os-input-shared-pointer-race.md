---
id: B-os-input-shared-pointer-race
title: "Two editors on one X display can interleave os_input motion and presses; ClickAt presses wherever the shared pointer is, so a click can land in the other editor's window"
status: DONE
severity: High
category: bug
tags: [drive, drive.click, os-input, xtest, x11, linux, shared-display, concurrency, silent-misdelivery]
encounters: 1
costly: 0
lastSeen: 2026-09-30T12:00:00Z
---

# os_input has no cross-process serialization on a shared display

UE 5.8 Linux, plugin `2580e7f4`. Found by code reading while discussing the multi-editor
wrong-window problem (`B-drive-click-os-input-foreign-x-window`); not yet reproduced live.

An X display has ONE core pointer. `FDriveOsInput::ClickAt`
(`Source/PinWright/Private/Handlers/Drive/DriveOsInput.cpp`) runs `MoveTo`, which sends a
~250 ms paced `XTestFakeMotionEvent` path. It then calls `FindForeignWindowAt(ScreenPos)` and
sends `XTestFakeButtonEvent` press, holds ~80 ms, and releases. The ownership gate checks the
window at the TARGET point. It does not check that the pointer is still AT the target point,
and nothing stops a second editor from injecting at the same time.

Scenario: editors A and B (two checkouts or agents) on `:0`, windows side by side, no overlap,
so each gate passes. Both call `drive.click {os_input:true}` within the same half-second:

- B's motion path runs while A is between "motion done" and "press". A's press goes to B's
  current pointer position, which can be inside B's window or a third application.
- B's motion during A's 80 ms hold turns A's click into a drag, and A's release lands elsewhere.

Neither call reports it: A's `ClickAt` returns true and the outcome is
`no_change_within_budget` or a wrong change.

**Expected:**
1. Hold a cross-process lock for the whole `MoveTo` / `ClickAt` sequence, keyed by display:
   `flock` on `$XDG_RUNTIME_DIR/pinwright-os-input-<DISPLAY>.lock` (fall back to `/tmp` with
   the uid in the name). Portable across GPUs and distros, needs no new link dependency.
2. Bounded wait (a few seconds). On timeout, refuse with a clear code, naming the pid that
   holds the lock (write it into the lock file).
3. Belt and braces: right before the press, `XQueryPointer` must equal the target point;
   otherwise refuse with `INPUT_FAILED` ("pointer moved by another client").
4. A future Windows `SendInput` path needs the same guarantee: a named mutex
   (`Local\pinwright-os-input`) plus a `WindowFromPoint` + `GetWindowThreadProcessId` ownership
   gate from day one.

The lock does not cover a human moving the real mouse or raw `xdotool` scripts; check 3 does.

## History
- `#1-filed-code-reading` `OPEN` reporter - Filed from analysis of `DriveOsInput.cpp` at `2580e7f4` after the foreign-window gate (`B-drive-click-os-input-foreign-x-window` #5) landed. The gate stops a press into a window that covers the target; it cannot stop two concurrent injectors sharing one pointer. Related: `F-os-input-private-display` (removes the sharing entirely where the GPU allows it).
- `#2-display-lock-and-prepress-check` `IN-REVIEW` developer - Linux/X11 fix in `Source/PinWright/Private/Handlers/Drive/DriveOsInput.{h,cpp}`. (1) New `FDriveOsInput::FDisplayLock`: `flock(LOCK_EX|LOCK_NB)` polled every 10 ms on `/tmp/pinwright-os-input-<uid>-<display>.lock` (`LockPathFor`, screen suffix dropped so `:0`/`:0.0` share it; fixed under /tmp, not `$XDG_RUNTIME_DIR`, so two editors launched with different environments still meet on one file; `O_NOFOLLOW`). The holder writes its pid into the file. `MoveTo` holds it for the motion; `ClickAt` holds ONE lock across motion, foreign-window re-check, pointer check, press, hold and release. Bounded wait `LockTimeoutSeconds` = 5 s, then refuses with the new registered code `OS_INPUT_BUSY` (payload `holder_pid`, `lock_file`), nothing injected. (2) Right before the press `ClickAt` calls `IsPointerAt` (`XQueryPointer`); off target refuses with the new registered code `POINTER_MOVED` (payload `x`,`y`,`pointer_x`,`pointer_y`), button not pressed. Chose a distinct code over `INPUT_FAILED` because the caller acts differently (pointer moved by a human/xdotool or clamped by a grab vs. a dead X connection). (3) The injection's error now reaches the caller: `FDriveActionCommon::FInject` takes a `FDriveInjectFailure&` (new struct in `DriveInput.h`, pre-filled with the old generic `INPUT_FAILED`), and `RunAction` sends its code/message/details; previously `drive.click`/`drive.hover` discarded the os_input error string and always answered the generic "Synthetic input injection failed" message (this also makes the existing post-motion foreign-window refusal report its real reason). Codes in `Handlers/ErrorCodes.h`; `os_input` param text in `DriveActionHandlers.cpp`; wiki `docs/wiki-src/drive.md` (drive.click os_input section). Tests in `Tests/Drive/TestDriveOsInput.cpp`: `PinWright.drive.os_input.DisplayLockPath` (pure), and Linux-only `PinWright.drive.os_input.DisplayLockExcludes` (two acquirers on one file: second times out with `bTimedOut`, names the holder pid, then a third acquires after release), `PinWright.drive.os_input.ClickWaitsForDisplayLock` (holds this display's real lock and asserts `ClickAt` refuses `OS_INPUT_BUSY` with `holder_pid`; needs no X display; costs ~5 s; with the lock wiring reverted it gets `INPUT_FAILED` and never presses because the target is off-screen), `PinWright.drive.os_input.PrePressPointerCheck` (real X pointer read: refusal for an off-screen target, pass for the pointer's own position; skips with `reason=no-x-display` when DISPLAY is absent). Not covered: the Windows `SendInput` half (expected item 4), tracked as `F-drive-os-input-windows`; hover does not run the pointer check (no press to misdeliver).
- `#3-verified-linux` `DONE` tester — Linux/X11 verified. A dedicated run with `DISPLAY=:0` (`Automation RunTests PinWright.drive.os_input`, offscreen) passed 7/7 with no skip markers, so `PrePressPointerCheck` read the real X pointer; `DisplayLockExcludes` and `ClickWaitsForDisplayLock` prove the per-display lock and the `OS_INPUT_BUSY` refusal. In the final full suite `dc867ac2` (offscreen, Linux Vulkan, UE 5.8, PinWright `5303edd1` on `bf3f2a9f`): 5460/5461 pass, the one failure is the host-only relay DNS error in `drive.observe.ScreenshotIncludesUmgAndMarksAtSurfaceLocalCoords`, unrelated the group passed 7/7 with `PrePressPointerCheck` skipped (`no-x-display`, the supervisor's editor has no DISPLAY). Windows half stays with `F-drive-os-input-windows`.
