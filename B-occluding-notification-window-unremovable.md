---
id: B-occluding-notification-window-unremovable
title: "drive.click TARGET_OCCLUDED by a persistent Notification window says 'Close or move that window', but no RPC can: editor.resize_window reports success while the size is unchanged, set_window_state is declined, and there is no close/dismiss verb"
status: OPEN
severity: Medium
category: bug
tags: [drive, drive.click, target-occluded, notification, toast, resize-window, set-window-state, render-offscreen, pie, silent-false-success, no-recovery-path]
encounters: 1
lastSeen: 2026-09-25T11:33:00Z
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
