---
id: B-drive-key-shift-f1-arrives-bare
title: "drive.key {key:F1, modifiers:shift} in PIE acts as a bare F1: the game viewport switches to wireframe instead of releasing the mouse"
status: OPEN
severity: Medium
category: bug
tags: [drive, drive.key, modifiers, pie, game-viewport, keyboard, silent-wrong-action]
encounters: 1
lastSeen: 2026-09-30T12:19:00Z
---

# Shift+F1 through drive.key arrives as a bare F1

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin `adb239fd`, visible editor, standalone
PIE on `L_PDS_Stadium` with a racing HUD that had put the viewport in `CapturePermanently` /
`LockOnCapture` (per `drive.input_state`).

To release the PIE mouse capture I sent `drive.key {key:"F1", modifiers:"shift", instance_name:"W_OverallUILayout"}`.
The call returned `outcome:"settled_changed"`. The capture stayed on (`drive.input_state` unchanged), and
the next `editor.screenshot` showed the game viewport in **wireframe**. F1 alone is the PIE wireframe view
mode hotkey, and Shift+F1 is the engine's "release mouse" chord. So the chord reached the viewport as a
bare F1. `drive.key {key:"F3"}` put the viewport back to lit.

Likely cause (from `B-drive-type-pie-editable-text-no-input`): `drive.key` sets modifiers only on the
key event's `FModifierKeysState` (`DriveInput.cpp`) and never sends a Shift key-down. The viewport's
Shift+F1 handler checks the viewport key state for a held Shift, so it only sees F1.

**Expected:** `modifiers` produces a real chord (modifier key-down, key, key-up, modifier key-up), so
handlers that query held-key state see it. If it cannot, the response says the modifiers were only set
on the event flags. The docs name the Shift+F1 capture release as a case that needs a real chord.

**Cost:** cheap here (one extra call to undo the wireframe), but the call changed editor view state
silently while reporting success.

## History
- `#1-shift-f1-became-wireframe` `OPEN` reporter - "Filed from the PDS QA #915 Time Attack repro in wt2. drive.key F1 with modifiers:shift switched the PIE viewport to wireframe instead of releasing the mouse capture. F3 restored lit. Workaround: Escape opened the game's pause menu, which dropped the capture."
