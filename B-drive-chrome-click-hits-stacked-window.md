---
id: B-drive-chrome-click-hits-stacked-window
title: "drive.click on editor_chrome with window_index injects at desktop coordinates, so a different window stacked at the same rect receives the click, and the call reports no_change_within_budget"
status: DONE
severity: Medium
category: bug
tags: [drive, drive.click, editor-chrome, window-index, slate-injection, overlapping-windows, no-change-within-budget, silent-wrong-target]
encounters: 1
lastSeen: 2026-09-24T09:05:00Z
---

# The addressed window is not the one clicked

## Symptom

After `editor.play` the editor had four floating windows at the identical rect `{x:812,y:430.5,w:1006,h:606}`:
`Message Log` (index 1), `Content Browser`, `BA Welcome Screen`, `PinWright Setup`. `drive.click
{surface:"editor_chrome", window_index:1, handle:"SWindow/.../SDockingTabWell[1]/SDockTab[0]/.../SButton[2]"}`
(the Message Log tab close button, observed at `{x:1032,y:442}`) closed `PinWright Setup` instead, then on
repeat `BA Welcome Screen`, then `Content Browser`; only the fourth call closed Message Log. The first three
returned `outcome:"no_change_within_budget", changed:false` because the settle loop watched window 1, which
indeed did not change. Nothing says the press landed on another window.

## Repro

UE 5.8 Linux, plugin `ba115afb`, a project whose editor restores several floating tabs at one position;
`editor.play`; `drive.list_windows`; `drive.click` a tab close button in the window with the lowest index.

## Ask

Either route the synthetic press to the addressed `SWindow` (bring it to front first), or report in the
response which window received the press when it is not the addressed one.

## Related

- `B-drive-geometry-window-space-injected-as-desktop` (IN-REVIEW) - also desktop-coordinate injection, but about a non-zero window origin; here the rect is right and a different window stacked at the same rect takes the press.

## History

- `#1-stacked-window-receives-click` `OPEN` reporter - Filed from a school lesson-attempt PIE session
  (Linux, host `/sdb-disk/src/unreal/unreal-fpv`, plugin `ba115afb`); reproduced on three editor restarts.
- `#2-gated-by-window-routing` `IN-REVIEW` developer - Not reproducible since plugin `61c243f5`
  (2026-09-25, "Gate drive pointer input on the target's own window", filed under
  `B-drive-click-misses-pie-game-viewport`; `ba115afb` predates it). `FDriveActionCommon::RunAction`
  resolves the handle inside the window `window_index`/`window_title` selects, takes that widget's own
  `SWindow`, moves the pointer to the target center and injects only once `FDriveInput::WindowUnderPoint`
  (the same `LocateWindowUnderMouse` the injection uses) names that window; otherwise, after 0.3 s, it
  refuses with `TARGET_OCCLUDED {occluding_window, occluding_window_type, recovery}` and presses nothing.
  `os_input` gets the same check via `TopWindowAtPoint` (`3cddd7da`). That is the ticket's second
  option (report the window) and stronger: no wrong-window press, no `no_change_within_budget`.
  Not bringing the addressed window to front: that changes editor state the caller did not ask for.
  Added the ticket's exact shape as a regression test: two top-most fixture windows at one identical
  rect, target addressed by `window_index` -
  `PinWright.drive.click_occlusion.StackedSameRectByIndexIsRefused`
  (`Source/PinWright/Private/Tests/Drive/TestDriveClickOcclusion.cpp`; fails if the routing gate is
  removed, since the press then lands on the cover and returns success). Wiki: `docs/wiki-src/drive.md`
  drive.click gate paragraph names same-rect stacked editor-chrome windows.
- `#3-verified-linux` `DONE` tester — Test commit fc64363c; gate from 61c243f5/3cddd7da (both on master). Passed non-skipped in run3/full and run3/xdrive: `PinWright.drive.click_occlusion.StackedSameRectByIndexIsRefused` (two top-most fixture windows at one identical rect, target addressed by `window_index`; the click is refused `TARGET_OCCLUDED` naming the covering window and nothing is pressed). Also passed: `OccludedTargetIsRefused` and `UncoveredTargetIsClicked` in the same suite. Acceptance: the ask's second option (report which window would receive the press) is met in a stronger form. No press lands on a different window, and there is no `no_change_within_budget` false outcome. Bringing the addressed window to front was declined on purpose (it would change editor state unasked); the ask allowed either option. Limit: the reporter's four restored floating tabs after `editor.play` were not replayed; the fixture reproduces the same-rect stacking shape.
