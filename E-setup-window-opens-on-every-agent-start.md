---
id: E-setup-window-opens-on-every-agent-start
title: "The 'PinWright Setup' window opens on every editor started by editor_start / editor_restart, over the PIE viewport, so the first drive.type / drive.click into PIE fails TARGET_OCCLUDED until an agent closes it by hand"
status: OPEN
severity: Low
category: ergonomic
tags: [setup-window, editor_start, editor_restart, pie, drive, occlusion, target-occluded, startup]
encounters: 1
lastSeen: 2026-09-25T12:21:00Z
---

# The setup window keeps reappearing in agent-started editors

## Symptom

UE 5.8, PDS (`unreal-fpv`), Linux. In this session the editor was restarted eight times through `editor_start` /
`editor_restart`, each time on an already configured project. A floating `PinWright Setup` window (about 1006x606,
centred) was open after the start. The first `drive.type {handle:"LastNameBox"}` into the PIE login form failed:
`[TARGET_OCCLUDED] Element 'LastNameBox' is covered at (1066, 806) by window 'PinWright Setup'`. `wmctrl -c
'PinWright Setup'` fixed it. The occlusion check itself worked and the error named the window, which made the fix
quick; the problem is the window reopening at all.

## Expected

Don't open the setup window when the project is already set up, or at least not in an editor that `editor_start` /
`editor_restart` launched for an agent (an automation start). If it must open, open it docked or behind the level
editor, not over the viewport.

## Workaround

Close it after every start (`wmctrl -c 'PinWright Setup'`, or a drive.click on its close button).

## History
- `#1-reopens-on-agent-start` `OPEN` reporter - Hit during PIE verification of the acceptance service pupil (PDS QA #1044). Related: `B-drive-chrome-click-hits-stacked-window` mentions the same window in a stacked-window click.
