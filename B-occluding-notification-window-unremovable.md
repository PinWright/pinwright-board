---
id: B-occluding-notification-window-unremovable
title: "drive.click TARGET_OCCLUDED by a persistent Notification window says 'Close or move that window', but no RPC can: editor.resize_window reports success while the size is unchanged, set_window_state is declined, and there is no close/dismiss verb"
status: IN-REVIEW
severity: Medium
category: bug
tags: [drive, drive.click, target-occluded, notification, toast, resize-window, set-window-state, render-offscreen, pie, silent-false-success, no-recovery-path]
encounters: 2
lastSeen: 2026-09-30T10:00:00Z
---

# An occluding editor toast cannot be cleared, so drive.click into PIE dead-ends

UE 5.8 Linux, host `unreal-fpv-wt2`, editor started by
`editor_start {visible:false, map:"/Game/System/FrontEnd/Maps/L_Core"}`
(`-RenderOffScreen -unattended`, main frame 1280x720). PIE in the level viewport,
LAN-hosted room on `L_PDS_Stadium`.

`drive.click` on a PIE HUD button
(`.../W_CommonTrackControls/BecomeSpectatorButton/SCommonButton`, geometry 912,588 95x22)
returned `TARGET_OCCLUDED`, first naming `ULTIMATE BLUEPRINT GENERATOR` and, after that
window was shrunk with `editor.resize_window` (which worked), naming window `''` — the
editor's persistent "It is recommended that you set up a valid Shared DDC Path" toast
(`drive.list_windows` index 4, `type: Notification`, 913,530 352x138). The error text
says "Close or move that window", but no available verb can:

- `editor.resize_window {window_index:4, width:10, height:10}` answered **without an
  error or warning**: `requestedClientSize {10,10}`, `clientSize {346,132}`,
  `windowSize {352,138}`. The window did not change, and the response does not flag
  the mismatch (compare `editor.set_window_state`, which adds a `warning` when the
  measured state differs from the request).
- `editor.set_window_state {state:"minimized"}` on the other occluder was declined
  by the window manager (`stateMatchesRequest:false`, as its doc predicts offscreen).
- There is no `editor.close_window` / notification-dismiss verb, and `drive.observe`
  on the toast exposes only an `Open Settings` button (no close/dismiss control).

Workaround used: bypass the click entirely with `object.call_function` on the player
controller (`Server_SwitchToSpectator`), which is not faithful input.

**Expected:** either a verb that closes/dismisses a top-level window or Slate
notification (at least `type: Notification`), or `TARGET_OCCLUDED` recovery text that
names a working remedy; and `editor.resize_window` should report a warning (or fail)
when the measured client size differs from the requested one.

## History
- `#1-shared-ddc-toast-blocks-click` `OPEN` reporter - Filed from the QA-1028 ESC-menu voice-row session (PDS wt2): persistent Shared DDC toast occluded a PIE HUD button; resize_window silently kept its size; no close verb exists.
- `#2-offscreen-plugin-windows-and-toast` `OPEN` reporter - Second encounter, same host (`unreal-fpv-wt2`), UE 5.8.2 Linux, plugin `2580e7f4`, editor started offscreen, listen-server PIE with 1 client. The editor's own plugin windows (`BA Welcome Screen`, `Message Log`, `PinWright Setup`, `ULTIMATE BLUEPRINT GENERATOR`) stayed open over the PIE surface and blocked clicks. With no X display, `wmctrl`/`xdotool` could not reach them, so each had to be shrunk with `editor.resize_window`, which worked for them. A Notification window at (273,290) **could not be resized at all**, the same dead end as `#1`. Related: `E-setup-window-opens-on-every-agent-start` (one of these windows) and `B-drive-os-input-own-window-occlusion` (the same window stack in visible editors). A way to stop agent-started offscreen editors from opening these windows, or a close verb, would remove both workarounds.
- `#3-dismiss-verb-and-resize-warning` `IN-REVIEW` developer - New verb `editor.dismiss_notifications` (`Handlers/Editor/EditorNotificationHandler.cpp`, new error code `NO_NOTIFICATIONS` in `ErrorCodes.h`). It finds every visible notification window through `FSlateNotificationManager::GetWindows`, fades each `SNotificationItem` in it out with a zero-length fade (persistent toasts included), and waits up to 3 s for Slate to close the windows. It returns `{dismissed, requested, notifications[{text, closed}]}`, where `closed` is measured from Slate's visible-window list, plus a `warning` if any window stayed open. `TARGET_OCCLUDED` now carries `occluding_window_type` and `recovery`. For a `Notification` occluder, the message and `recovery` name `editor.dismiss_notifications {}` instead of 'Close or move'. `editor.resize_window` adds a `warning` when the measured client size differs from the request by more than 1 px, and names the dismiss verb for notification windows (`EditorWindowHandlers.cpp`). Wiki: `docs/wiki-src/editor.md` and `drive.md`. Test: `PinWright.editor.dismiss_notifications.ClosesPersistentToast` (a `bFireAndForget=false` toast must be reported `closed:true`, and its window must be gone).
- `#4-linux-verification` `IN-REVIEW` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Passed in w23-final: `PinWright.editor.dismiss_notifications.ClosesPersistentToast` (a non-fire-and-forget toast is reported `closed:true` and its window is gone), so the first Expected option, a verb that clears a notification window, is met. Not covered by any test: (1) `editor.resize_window` adding a `warning` when the measured client size differs from the request (no test exercises that path); (2) `TARGET_OCCLUDED` for a `Notification` occluder carrying `occluding_window_type` and a `recovery` naming `editor.dismiss_notifications` (the click_occlusion tests assert only `occluding_window` / `occluding_pid`). Needs: tests for the resize mismatch warning and for the notification recovery payload, or the offscreen Shared DDC toast repro.
