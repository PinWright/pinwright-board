---
id: B-drive-click-os-input-foreign-x-window
title: "drive.click os_input:true injects XTEST into whatever X window is on top, including another process's editor, and reports no_change_within_budget instead of TARGET_OCCLUDED"
status: OPEN
severity: High
category: bug
tags: [drive, drive.click, os-input, xtest, x11, linux, target-occluded, shared-display, foreign-window, window-stacking, silent-misdelivery]
encounters: 1
lastSeen: 2026-09-25T16:47:00Z
---

# os_input clicks land in a foreign window on a shared X display

UE 5.8 Linux, host `unreal-fpv-wt1` (editor PID 167813, `editor_start visible:true`,
display `:0`). A second, unrelated editor (`unreal-fpv` checkout, PID 194477, run by
another session) was visible on the same display with an identical maximized
geometry (70,27 2490x1413). When it restarted it went to the top of the X stacking
order, above the wt1 editor.

`drive.click {os_input:true}` on a PIE HUD button
(`.../W_HUD_Spectator_C_1/W_HUD_Common/W_CommonTrackControls/StartButton/SCommonButton`,
desktop geometry 1841,1325 192x45) returned
`{"outcome":"no_change_within_budget","input_path":"os_x11","changed":false}`.
Straight afterwards, `xdotool getmouselocation` on `:0` reported
`x:1937 y:1347 window:58720364`, and that window belongs to PID 194477, the other
editor. The wt1 log shows no handler trace for the click. So the real button press
went into another process's editor. After `xdotool windowactivate`/`windowraise` on
the wt1 main window, the same `drive.click` call worked.

`drive.click` already returns `TARGET_OCCLUDED` when one of the editor's own Slate
windows covers the target (see `B-occluding-notification-window-unremovable`). The
OS path has no matching check against foreign top-level X windows.

**Expected:** before sending XTEST events, `os_input` checks that the top-level X
window at the target point belongs to this editor process (`XQueryPointer` or
`XTranslateCoordinates` on the root, plus `_NET_WM_PID`). If it does not, the call
either fails with `TARGET_OCCLUDED`, naming the foreign window and PID, or raises the
editor's window when a flag asks for it. It must never deliver a real click to
another application.

**Why it matters:** on a shared display the misdelivered click is a real mouse press
in someone else's app. That is a correctness problem for the verification, and a
safety problem for the other session.

## History
- `#1-filed-shared-display` `OPEN` reporter - "Filed from the QA #949 analyzer timeline PIE verification in wt1. Seen once: an os_input StartButton click went to another checkout's editor window (PID 194477) stacked above the target editor. Workaround: raise your own editor window with `xdotool windowactivate --sync <wid>` before each os_input call, then check `xdotool getmouselocation` names a window of your own PID."
