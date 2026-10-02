---
id: B-drive-os-input-own-window-occlusion
title: "drive.click os_input:true skips the TARGET_OCCLUDED gate, so the editor's own mapped child windows (BA Welcome Screen, Message Log, PinWright Setup, ULTIMATE BLUEPRINT GENERATOR, notification toasts) over the PIE viewport silently swallow XTEST clicks; agents must xdotool-move them first"
status: IN-REVIEW
severity: Medium
category: bug
tags: [drive, drive.click, os-input, xtest, x11, linux, target-occluded, editor-child-windows, pie, workaround]
encounters: 5
lastSeen: 2026-09-30T13:25:00Z
---

# os_input clicks are not checked against the editor's own windows

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin clone `61c243f5`. The editor was started
with `editor_start {visible:true}` on `:0` (PID 3228606). Straight after the start, `xdotool search --pid`
listed these mapped, viewable top-level windows of that editor, all above the maximized main window
(70,27 2490x1413):

- `BA Welcome Screen` and `Message Log`, both at 70,27 646x366
- `PinWright Setup` at 812,431 1006x606
- `ULTIMATE BLUEPRINT GENERATOR - BluepintsLab(Roly Dev)` at 992,464 646x540
- two unnamed windows, 352x138 and 352x157, at 2193,1073 and 2193,1231 (notification toasts)

Several PIE targets sit under these rects. For example, the Race preselect tile `Button[1]` at 391,585
661x372 overlaps `PinWright Setup`. The Slate path has refused such clicks with `TARGET_OCCLUDED` since
`B-drive-click-misses-pie-game-viewport` #8, but that note also says "`os_input` and retainer virtual
windows are exempt". An os_input click therefore goes to whichever X window is on top at that point, here
an invisible editor child, and the call reports an ordinary settle outcome. A 2026-09-17 wt1 session saw
every `drive.click os_input:true` into a PIE popup do nothing for this reason (agent memory "Real-mouse
XTEST recipe"). In this session I applied the workaround before the first os_input click, so the failure
itself was not re-reproduced. The window stacking that causes it was confirmed.

**Workaround:** `xdotool windowsize <id> 100 50` then `xdotool windowmove <id> 5000 5000` for each of the
editor's own child windows (mutter clamps a bare move). The toasts snap back to the bottom-right corner
by themselves.

**Expected:** before sending XTEST events, os_input checks the top-level X window at the target point. If
it is not the target's own native window, whether it is another window of this editor or a foreign one
(see `B-drive-click-os-input-foreign-x-window`), refuse with `TARGET_OCCLUDED` and name the window. Better
still, give `editor_start` a way to stop agent-started editors from mapping these windows over the viewport
(see `E-setup-window-opens-on-every-agent-start`).

## History
- `#1-filed-os-input-exempt-gate` `OPEN` reporter - Filed during a PDS localization check in wt2 (TrackLoaderUtilWidget launch-failure popup). Six own editor windows were stacked over the PIE viewport at startup; I moved them with xdotool before the first os_input click. Cheap in this session, because the workaround was known from an earlier session.
- `#2-three-editor-starts-same-stack` `OPEN` reporter - Same stack on three editor starts, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`, editor PIDs 3049871, 3147326 and 3256675. Each start mapped `BA Welcome Screen` and `Message Log` at 70,27 646x366, `PinWright Setup` at 812,431 1006x606, and two or three unnamed 352-px toasts at the bottom right, all above the maximized main window and over the PIE viewport, where the map-editor palette and the objects I had to click are. I shrank and moved them with xdotool before every real-mouse batch, so no click was lost. On the third start one toast had already closed between `xdotool search` and `windowsize`, so the move failed with `BadWindow`. Cheap, but a manual step every time.
- `#3-same-stack-two-more-starts` `OPEN` reporter - Same window stack in a separate wt2 session (QA repro + fix verification for the DESERT NTCN launch bug, plugin clone `61c243f5`). On both of my `editor_start {visible:true}` runs (PIDs 3087854 and 3178369), `xdotool search --onlyvisible --pid` listed `BA Welcome Screen` and `Message Log` at 70,27 646x366, `PinWright Setup` at 812,431 1006x606, `ULTIMATE BLUEPRINT GENERATOR` at 992,464 646x540 and the two toasts at 2193,1073 / 2193,1231, all mapped over the maximized main window. `AnonymousLoginButton` (984,986) and the Race preselect `Button[1]` (391,585 661x372) fall inside `PinWright Setup`'s rect. I shrank and moved all six before the first `drive.click {os_input:true}` on each start, so, as in #1, the swallowed click was not re-reproduced here; with the windows moved, every os_input click landed (`xdotool getmouselocation` named the main editor window 54526060). Cheap: two xdotool loops.
- `#4-rates-button-swallowed-wt1` `OPEN` reporter - Re-reproduced the failure itself (not just the stacking). UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `adb239fd`, editor PID 984962 (`editor_start {mode:visible}`, main window 70,27 900x640). `drive.click {os_input:true}` on the pause menu's `W_PauseMenu/RatesButton` (desktop 657,350 74x17) returned `input_path:"os_x11"`, `outcome:"no_change_within_budget"`, no error, and the Rates page did not open; `xdotool getmouselocation` put the pointer on window 88080494 = `BA Welcome Screen` (70,27 646x366, same PID), with `Content Browser` and `Message Log` at the same rect. After `windowsize 100 50` + `windowmove` on those plus `PinWright Setup` and `ULTIMATE BLUEPRINT GENERATOR`, the identical call returned `settled_changed` and the page opened. Clicks outside the 646x366 rect (the HUD `Меню` button at 705,595) had worked before the move, which makes the failure look target-specific. Cost: one wasted click plus a screenshot and an X-window diagnosis.
- `#5-two-more-starts-wt1` `OPEN` reporter - Same stack on two more `editor_start`/`editor_restart {mode:visible}` runs in wt1 (PIDs 1069537, 1087591, plugin `adb239fd`). `BA Welcome Screen`, `Message Log` and `Content Browser` were at 70,27 646x366; `PinWright Setup` and `ULTIMATE BLUEPRINT GENERATOR` were mapped as well. The xdotool `windowsize` + `windowmove` workaround was applied before any os_input click each time. Cost: a script step per start.
- `#6-slate-order-gate-for-os-input` `IN-REVIEW` developer - `os_input` now gets the Slate window-order check that the Slate path has, without its wait. Before any X event, `FDriveActionCommon::RunAction` (`Handlers/Drive/DriveActionCommon.cpp`) asks `FDriveInput::TopWindowAtPoint` for the top window at the target's center by Slate's own order, which Slate keeps in step with activation, with top-most windows such as toasts above. If that is not the target's window, the call is refused with `TARGET_OCCLUDED` and the same payload as the Slate path (`occluding_window`, new `occluding_window_type` and `recovery`, no `occluding_pid`), and nothing is injected. This check runs before the existing foreign-X-window gate. The `os_input` param doc (`DriveActionHandlers.cpp`) and `docs/wiki-src/drive.md` are updated. Not done: X stacking of this editor's own windows is not read separately (Slate order is the gate), and `ClickAt`'s re-check after the motion still covers only foreign windows. Test: `PinWright.drive.click_occlusion.OsInputOwnWindowIsRefused`, a covered fixture clicked with `os_input:true` that must be refused naming the cover with no `occluding_pid`. It skips with `no-x-display` where os_input is unavailable.
