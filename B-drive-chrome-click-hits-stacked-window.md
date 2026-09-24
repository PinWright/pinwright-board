---
id: B-drive-chrome-click-hits-stacked-window
title: "drive.click on editor_chrome with window_index injects at desktop coordinates, so a different window stacked at the same rect receives the click, and the call reports no_change_within_budget"
status: OPEN
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
