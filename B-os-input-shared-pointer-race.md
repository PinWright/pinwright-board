---
id: B-os-input-shared-pointer-race
title: "Two editors on one X display can interleave os_input motion and presses; ClickAt presses wherever the shared pointer is, so a click can land in the other editor's window"
status: OPEN
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
