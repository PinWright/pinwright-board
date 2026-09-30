---
id: F-drive-os-input-windows
title: "drive.click / drive.hover os_input is Linux X11 only; add a Windows SendInput backend so real OS pointer input and capture/lock repros work on Windows"
status: OPEN
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
