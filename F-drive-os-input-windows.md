---
id: F-drive-os-input-windows
title: "drive.click / drive.hover os_input is Linux X11 only; add a Windows SendInput backend so real OS pointer input and capture/lock repros work on Windows"
status: IN-REVIEW
severity: High
category: feature
tags: [drive, os-input, windows, sendinput, dpi, drive.click, drive.hover, mouse-capture, compare-page]
encounters: 1
---

# os_input has no Windows backend

`os_input:true` on `drive.click` / `drive.hover` injects real OS pointer events, so input takes the OS
route (mouse capture, `LockOnCapture` confinement, relative mode, cursor hide) instead of Slate
synthesis. It is the only path that reproduces capture/lock bugs. Today it exists only on Linux X11:

- `DriveOsInput.cpp:70-71` resolves `XTestFakeMotionEvent` / `XTestFakeButtonEvent` via dlsym; `MoveTo`
  and `ClickAt` are `#if PLATFORM_LINUX`.
- Off Linux, `FDriveOsInput::IsAvailable` (`DriveOsInput.cpp:174`) returns "os_input is Linux/X11-only",
  and `ResolveOsInput` (`DriveActionHandlers.cpp:75-89`) refuses with `INVALID_ARGUMENT`. So on Windows,
  the primary dev platform, os_input is always refused.
- `drive.input_state` (`DriveInputStateHandler.cpp:89-120`: cursor captor, `mouse_capture_mode`,
  `mouse_lock_mode`, `hide_cursor_during_capture`) already works on every platform. Only the injection
  half is missing.
- There is no os_input on `drive.drag`, `drive.scroll` or `drive.key`. The dispatcher allowlist
  (`DRIVE_OS_INPUT_PARAM`, `DriveActionHandlers.cpp:60-72`) declares it only on click/hover.
- Tests: `Tests/Drive/TestDriveOsInput.cpp` covers only the pure helpers (motion path, button mapping).
  No test injects.

pinwright.com/compare grades PinWright `partial` on "Real OS pointer input and mouse-capture diagnosis",
because os_input is "Linux X11 and mouse only" (`compare-data/data/compare-final.json`, pinwright cell,
evidence `DriveOsInput.cpp:70-71`, `:174`). The grade becomes `yes` when Windows ships.

**Ask:** a Win32 branch in `FDriveOsInput` behind the same `IsAvailable` / `MoveTo` / `ClickAt` API,
with the same 24-step motion path and ~80 ms hold, using `SendInput` (`MOUSEEVENTF_MOVE | ABSOLUTE |
VIRTUALDESK`, then button down/up). `input_path` reports `"os_win32"`.

**Design constraints:**
- **Coordinates.** Map the target to screen pixels through the owning editor or PIE `SWindow`
  (`GetPositionInScreen` / native HWND rect), not raw cached geometry. `B-drive-geometry-window-space-injected-as-desktop`
  (IN-REVIEW) shows `geometry.absolute` can be window-relative and only matches desktop pixels when the
  window sits at (0,0). Normalise to 0..65535 over `SM_XVIRTUALSCREEN`..`SM_CXVIRTUALSCREEN`.
- **DPI.** Slate absolute coords are physical pixels. The editor is per-monitor DPI aware, so in-process
  `GetSystemMetrics` returns physical pixels. Verify on a scaled monitor (150%) and on a secondary
  monitor with a negative origin. `E-drive-wiki-os-input-injection-notes` records the 1707x960 vs
  2560x1440 trap for unaware injectors.
- **Foreground.** Injected input goes to whatever window is under the cursor or in the foreground.
  `SetForegroundWindow` is refused when the calling process is not foreground (foreground lock), and
  UIPI blocks injection into higher-integrity windows. Before pressing, bring the target window forward
  (`FGenericWindow::BringToFront` / `HACK_ForceToFront`) and confirm it with `GetForegroundWindow`.
  Also run a `WindowFromPoint` + `GetWindowThreadProcessId` guard: the point must be owned by this
  process's target HWND. If either fails, return a typed refusal (`TARGET_OCCLUDED` naming the owner,
  or a foreground-lock reason) and send nothing. The X11 gaps in `B-drive-click-os-input-foreign-x-window`
  and `B-drive-os-input-own-window-occlusion` must not be repeated here.
- **Pacing.** The X11 path sleeps on the calling thread. On Win32, if that is the game thread, the
  `WM_MOUSEMOVE`s queue and are pumped as one burst after the handler returns, which defeats the
  hand-like path. Check that the app sees the paced sequence (pump between steps, or run the gesture
  off the game thread).
- **Offscreen/headless.** `editor_start visible:false` (`-RenderOffScreen`), `-nullrhi`, and commandlet
  runs have no real window. `IsAvailable` returns a typed reason ("no visible window; start the editor
  with visible:true") instead of injecting blind. See `E-os-input-headless-editor-undocumented` for the
  X11 twin.
- **Scope order.** Ship click and hover first. The same backend covers drag/scroll os_input once
  `F-drive-os-input-point-gestures` adds them. Win32 keyboard `SendInput` may work where XTEST keys did
  not, but it is a separate follow-up: `drive.key` has no os_input today.

**Acceptance:**
- Live drive automation tests exercise `drive.click` and `drive.hover` with `os_input:true` on Windows
  in a visible editor. They assert `input_path == "os_win32"`, that the target actuated, and that
  `drive.input_state` shows the expected captor/lock change on a capturing PIE viewport.
- In offscreen/headless runs the same tests skip cleanly and log the reason. They must not fail and
  must not pass vacuously.
- `docs/wiki-src/drive.md` platform notes (lines 84-86 and the `drive.hover` entry at 108 say
  "Linux/X11 only") describe the Windows backend, its foreground and DPI rules, and the headless refusal.
  The `DRIVE_OS_INPUT_PARAM` description is updated to match.
- Verified on UE 5.8, Windows.

## History
- `#1-windows-backend-requested` `OPEN` reporter - Filed at the user's direction. Verified against plugin `adb239fd`: `DriveOsInput.cpp:70-71` XTEST-only, `:174` refuses off Linux, `ResolveOsInput` maps that to `INVALID_ARGUMENT`, and `drive.input_state` is already cross-platform. The compare page's `partial` on the OS-pointer row rests on this gap. Severity High: hard blocker with no in-tool workaround on the primary platform (the only fallback is an external DPI-aware `SendInput` script), on the row that differentiates PinWright.
- `#2-win32-backend-uncompiled` `IN-REVIEW` developer — **The Windows path is UNCOMPILED and UNTESTED: it was written on a Linux host, which cannot build or run Win32 code. Nothing below has been observed on Windows.** Only the Linux build was compile-checked. New `Handlers/Drive/DriveOsInputWindows.{h,cpp}` (whole body `#if PLATFORM_WINDOWS`, Windows headers via `Windows/AllowWindowsPlatformTypes.h` / `HideWindowsPlatformTypes.h`). `DriveOsInput.cpp` keeps the one `FDriveOsInput` API and delegates on Windows (`#elif PLATFORM_WINDOWS`) for `IsAvailable`, `FindForeignWindowAt`, `DisplayLockPath`, the `FDisplayLock` ctor/dtor (new `void* Mutex` member, `IsHeld` = fd or mutex), `IsPointerAt`, `MoveTo`, `ClickAt`, and the point-gesture primitives `BeginGesture` / `SendMotion` / `SendButton`, so drag/scroll gestures get the Win32 backend too. `ProbeGrabs` stays unimplemented on Windows (`os_grab: null`). Behavior: (1) the same 24-step `ComputeMotionPath` sent as `SendInput` `MOUSEEVENTF_MOVE|ABSOLUTE|VIRTUALDESK`, normalized with the new pure helper `FDriveOsInput::NormalizeVirtualDeskCoord` (ceil inversion of Windows' `Origin + (n*Extent)>>16`, origin `SM_XVIRTUALSCREEN` so negative origins are covered). No DPI conversion: the per-monitor-aware editor's Slate coords, `GetSystemMetrics`, `GetCursorPos` and `WindowFromPoint` all use physical pixels. (2) Pacing: the X11 timings, and after each sleep `FSlateApplication::PumpMessages()` on the game thread. In the main loop `GPumpingMessagesOutsideOfMainLoop` is false, so `FWindowsApplication::DeferMessage` processes each pumped `WM_MOUSEMOVE` / `WM_INPUT` immediately and Slate sees a paced sequence rather than one coalesced move. (3) Lock: `CreateMutexW("Local\\PinWright-os-input")` + `WaitForSingleObject(5 s)`; `WAIT_ABANDONED` counts as acquired; timeout gives `OS_INPUT_BUSY` with `holder_pid: 0` (a mutex does not record its owner). The mutex is recursive per thread (`ponytail:` note). (4) `ClickAt`: lock, then `WindowFromPoint`→`GetAncestor(GA_ROOT)`→`GetWindowThreadProcessId` must be this pid (else `TARGET_OCCLUDED` with occluding_window / _id / _pid). If this process is not foreground it calls `SetForegroundWindow(root)` (what `HACK_ForceToFront` does) and re-checks at process level, so non-activating Slate popups pass; a refusal gives the new `ERR_FOREGROUND_LOCKED` (foreground_window, foreground_pid). Then the motion, the foreign-window and foreground re-check, the `GetCursorPos` check (`POINTER_MOVED`, which also covers a `ClipCursor` / `LockOnCapture` clamp), and the button down, hold, up (the up is always attempted, and swapped buttons are honored via `SM_SWAPBUTTON`). A `SendInput` that inserts 0 events (UIPI) gives `INPUT_FAILED`. (5) `IsAvailable` refuses with a reason for a commandlet, `-RenderOffScreen` (null platform app), `-nullrhi`, uninitialized Slate, an empty virtual screen, or `OpenInputDesktop` failing (locked or disconnected). `ResolveOsInput` maps that reason to `INVALID_ARGUMENT`. Label: new `FDriveOsInput::InputPathLabel()` ("os_win32" / "os_x11") replaces the hard-coded `"os_x11"` in `DriveActionHandlers.cpp` (click, hover) and in the three os-path gates in `DriveActionCommon.cpp`; the Wayland session note stays X11-only. `SendOsOccluded` now says "OS window". `DRIVE_OS_INPUT_PARAM` and `docs/wiki-src/drive.md` (input_path values, a new **Windows** paragraph, the platform sentence, the `drive.hover` entry) were updated, plus a CHANGELOG line marked untested. Tests in `Tests/Drive/TestDriveOsInputWindows.cpp`: `PinWright.drive.os_input.VirtualDeskNormalization` runs on every platform; `Win32LockExcludes`, `Win32ClickWaitsForLock`, `Win32ForeignPointRefused`, `Win32PrePressPointerCheck`, `Win32LiveClickActuates`, `Win32LiveHoverMovesPointer` (all `PinWright.drive.os_input.*`) are `#if PLATFORM_WINDOWS`, and all but the lock test skip with `os-input-unavailable` plus the IsAvailable reason when headless. The existing `PinWright.drive.click_occlusion.OsInputOwnWindowIsRefused` also starts running on Windows. **Not done:** the acceptance item "drive.input_state shows the expected captor/lock change on a capturing PIE viewport" has no automated test; it needs a manual pass. **What a Windows tester must run:** (a) build PDSEditor Win64 Development and fix any compile errors in `DriveOsInputWindows.cpp`, `DriveOsInput.cpp`, `TestDriveOsInputWindows.cpp`. (b) In a visible editor (`mode: "visible"`), on an unlocked desktop with the editor able to take the foreground, run `editor_run_tests` with filter `PinWright.drive.os_input+PinWright.drive.click_occlusion`. All must pass with no skip markers, and the `os_input` live tests must report `input_path == "os_win32"`. (c) Run the same filter in `mode: "offscreen"`. The Win32 live tests must skip with `reason=os-input-unavailable` naming `-RenderOffScreen`, and nothing may fail. (d) By hand with `mcp__pinwright__call`: `drive.click {os_input:true}` on a button on a 150%-scaled monitor and on a secondary monitor left of the primary (negative origin). Each click must land, and no `POINTER_MOVED` may come back with `pointer_x/y` off by one. An off-by-one there means the normalization model is wrong. (e) With buttons swapped in Windows mouse settings, `button:"left"` must produce a left click. (f) PIE with a capturing viewport (`CapturePermanently`, `LockOnCapture`): `drive.input_state`, then `drive.click {os_input:true}` into the viewport, then `drive.input_state`. The captor and lock must change, and moves must arrive as a paced sequence (watch `WM_INPUT` deltas or a mouse-delta print), not one jump. (g) With another app in front and the editor in the background, `drive.click {os_input:true}` must either bring the editor forward and click, or return `FOREGROUND_LOCKED` with nothing pressed. (h) Two editors clicking at once: the second must wait or return `OS_INPUT_BUSY`. Files: `Source/PinWright/Private/Handlers/Drive/{DriveOsInputWindows.h,DriveOsInputWindows.cpp,DriveOsInput.h,DriveOsInput.cpp,DriveActionCommon.cpp,DriveActionCommon.h,DriveActionHandlers.cpp}`, `Handlers/ErrorCodes.h` (`ERR_FOREGROUND_LOCKED`; `TARGET_OCCLUDED` comment), `Tests/Drive/TestDriveOsInputWindows.cpp`, `docs/wiki-src/drive.md`, `CHANGELOG.md`.
- `#3-linux-verification` `IN-REVIEW` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Linux side only: `PinWright.drive.os_input.VirtualDeskNormalization` (the shared pure helper) passed in w23-final and w23-xfinal, and the Linux build compiles with the Windows delegation in place. The Win32 backend itself is uncompiled and untested (developer #2), and every Acceptance bullet requires UE 5.8 on Windows. Needs: the Windows tester checklist (a)-(h) in #2.
