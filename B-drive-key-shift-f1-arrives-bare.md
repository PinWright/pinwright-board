---
id: B-drive-key-shift-f1-arrives-bare
title: "drive.key {key:F1, modifiers:shift} in PIE acts as a bare F1: the game viewport switches to wireframe instead of releasing the mouse"
status: IN-REVIEW
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
- `#2-real-modifier-chord` `IN-REVIEW` developer - "Root cause: two places dropped the modifier. (1) `FSceneViewport::OnKeyDown` keeps only the key in its `KeyStateMap` and `UPlayerInput` only learns Shift from a LeftShift key event, which drive.key never sent. (2) `UGameViewportClient::InputKey` hands the editor's play-world chords (Shift+F1 `GetMouseControl`) `FSlateApplication::GetModifierKeys()`, the platform state (SDL_GetModState on Linux), never the event's flags; F1's DebugExecBinding is `bIgnoreShift=True`, so it fell through to wireframe. Fix in the shared funnel `FDriveInput::PressKeyReportingHandled` (all drive.key / web key / editor.simulate_input editor-route callers): modifiers are now a real chord, LeftControl/LeftAlt/LeftShift/LeftCommand key-downs before the key's down edge and key-ups in reverse after its up edge (action down/up split them), and on Linux the SDL modifier bits are held for the chord via the `SDL_GetModState`/`SDL_SetModState` symbols ApplicationCore exports (dlsym, no new link dependency). Windows keeps its platform modifier array private, so there the GetModifierKeys()-only path (PIE Shift+F1) still sees a bare key; documented on the drive wiki page. Files: Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveInput.cpp, DriveInput.h (comment), Tests/Drive/TestDriveInputModifierChord.cpp (new), docs/wiki-src/drive.md (drive.key section). Test: `PinWright.drive.input.ModifierChordRealKeyDowns` (front-of-queue Slate preprocessor records and consumes every key edge; asserts modifier down/key down/key up/modifier up order for shift/ctrl/alt, event flags, and on Linux the platform modifier state during the key-down and its release after). Not yet verified live in PIE."
- `#3-offscreen-null-platform-app` `IN-REVIEW` developer - "Suite run (offscreen, both without DISPLAY and with DISPLAY=:0) failed the platform-hold assertions of ModifierChordRealKeyDowns. Cause: under -RenderOffScreen `FLinuxPlatformApplicationMisc::CreateApplication` returns `FNullApplication`, whose `GetModifierKeys()` is a constant empty state; SDL_SetModState worked but nothing read it (FLinuxApplication::GetModifierKeys does call SDL_GetModState, through the global copy ApplicationCore exports). The hold cannot be made there, so it is now reported, not assumed: `PressKey`/`PressKeyReportingHandled` take an optional `bOutPlatformModifiersHeld`, read back from `GetModifierKeys()` at the key's down edge; drive.key with modifiers on a press/down edge returns `modifiers_platform_held` plus a `warning` when false (new optional `InjectFields` param on `FDriveActionCommon::RunAction` carries injection-time fields into the success response). Tests: `PinWright.drive.input.ModifierChordRealKeyDowns` now asserts on every host that the reported flag equals the platform's answer; the hold itself moved to new `PinWright.drive.input.ModifierChordPlatformStateHeld`, which skips with `reason=platform-modifier-state-unavailable` unless Linux without -RenderOffScreen (needs a visible-mode run to measure). Files: DriveInput.cpp/.h, DriveActionCommon.cpp/.h, DriveActionHandlers.cpp (drive.key), TestDriveInputModifierChord.cpp, docs/wiki-src/drive.md."
