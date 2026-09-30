---
id: B-drive-click-os-input-foreign-x-window
title: "drive.click os_input:true injects XTEST into whatever X window is on top, including another process's editor, and reports no_change_within_budget instead of TARGET_OCCLUDED"
status: IN-REVIEW
severity: High
category: bug
tags: [drive, drive.click, os-input, xtest, x11, linux, target-occluded, shared-display, foreign-window, window-stacking, silent-misdelivery]
encounters: 5
costly: 1
lastSeen: 2026-09-30T12:30:00Z
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
- `#2-second-hit-peer-splash-and-main` `OPEN` reporter - Second encounter, UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin clone `61c243f5`, my editor PID 3228606 (`editor_start visible:true` on `:0`, maximized at 70,27 2490x1413). Mid-verification, another session started a wt1 editor (PID 3241515). First its 1200x600 startup splash (`Unreal Editor`, X window 71303220, at 715,433) and then its maximized main window, at the same rect as mine, went above my editor. `drive.click {os_input:true, wait_for:{widget_absent...}}` on a PIE location tile at (1652,697) returned `outcome:"timeout"`, `input_path:"os_x11"`, with no screen change. `xdotool getmouselocation` then named window 71303220, owned by the peer's PID 3241515. So a real click went into another session's editor again, and nothing in the response says so. I did not use the #1 raise-own-window workaround, because while my window is raised the peer's own os_input clicks would land in my editor. I finished the flow on the Slate path instead. Cheap (a few calls), but it is a cross-session safety problem.
- `#3-raw-xtest-into-peer-fullscreen-game` `OPEN` reporter - Third encounter, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`, my editor PID 3049871 on `:0`. I ran a batch of real-mouse clicks in PIE with raw `xdotool`, because `drive.click os_input` cannot target a 3D actor in the game viewport (see `F-drive-os-input-point-gestures`), so it would not have helped here. Partway through the batch, a peer session's packaged game (`PDSGame`, PID 3043884, wt1 scratch user dir) held a fullscreen 2560x1440 window above my editor. `xdotool getmouselocation` named window 54526052, owned by that PID. About 50 real presses and drags, a few of them right-button, went into the peer's game before I noticed, because my selection readback kept returning the unchanged state. Later the same day a wt1 editor (PID 3241515, maximized at the same 70,27 2490x1413 rect) sat above my re-run editor for tens of minutes, and I had to wait instead of clicking. I added my own guard (abort unless `getmouselocation` names my main window). A built-in os_input guard of the kind this ticket asks for would have refused the first click. Costly: one 9-trial batch lost and re-run, plus misdelivered input to another session.
- `#4-guarded-xdotool-workaround-wt1` `OPEN` reporter - Fourth encounter (hazard, not a misdelivery), UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`, my editor PID 3241515 on `:0`, with the wt2 editor and a freshly started main-checkout editor (PID 3256675) sharing the display. Before each real click I ran a guard (`xdotool mousemove X Y; xdotool getmouselocation --shell; xdotool getwindowpid $WINDOW` must equal my PID, else refuse). It refused twice: once over the main-checkout editor's `PinWright Setup` window (812,431 1006x606) and once over the wt2 main window stacked above my viewport; `xdotool windowraise <my main window>` before each click fixed it. `drive.click os_input` could not be used anyway (see the multi-PIE drive targeting ticket), but had it been, the same clicks would have gone into peer editors. Not costly (about four extra calls per click).
- `#5-x-ownership-gate` `IN-REVIEW` developer - Fixed in the plugin. `FDriveOsInput::FindForeignWindowAt` (`Source/PinWright/Private/Handlers/Drive/DriveOsInput.cpp`) walks the X tree at the target point with `XTranslateCoordinates` from the root down and reads each window's `_NET_WM_PID`. The deepest window that names a pid owns the point, so a WM frame carrying its own pid does not count (pure rule: `FDriveOsInput::IsPointOwnedBy`). A point owned by another pid, or by no named window, is foreign. `FDriveActionCommon::RunAction` (`DriveActionCommon.cpp`) runs that check for `input_path:"os_x11"` before anything is injected. On a hit it refuses with `TARGET_OCCLUDED` and details `occluding_window` (X title), `occluding_window_id` and `occluding_pid`, the same shape as the Slate-window gate. `FDriveOsInput::ClickAt` re-checks after the ~0.25 s motion and skips the press if a window was raised meanwhile. That case comes back as `INPUT_FAILED`. The new Xlib symbols are dlopen/dlsym'd like the existing ones, and the descent runs under a temporary no-op X error handler, so a window that vanishes mid-walk cannot trigger Xlib's exit-on-error. Where there is no X display (Wayland or headless), `IsAvailable` still refuses os_input with `INVALID_ARGUMENT`, as before. Test: `PinWright.drive.os_input.PointOwnership`. Docs: `docs/wiki-src/drive.md` (drive.click os_input section) and the `os_input` param description.
- `#6-fifth-hit-wt2-pre-fix-build` `IN-REVIEW` reporter - Fifth encounter, pre-fix build (not a regression): UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin `adb239fd` (does not contain `FindForeignWindowAt`), my editor PID 937718 (`editor_start visible:true`, main window 70,27 900x640 on `:0`). The wt1 editor (PID 984962) main window sat at the identical rect above mine. Three `drive.click {os_input:true}` calls on PIE HUD Start/Stop buttons (desktop 705,575 68x16) worked while mine was on top. A fourth one, on the lobby Start button, returned `outcome:"no_change_within_budget"`, `input_path:"os_x11"`. `xdotool getmouselocation` then named window 88080492 (`PDS - Unreal Editor`, PID 984962), so the press went into the peer's editor during its own QA run. I switched to the Slate path and keys for the rest of the flow. Not costly for me (a few calls); cross-session misdelivery again.
