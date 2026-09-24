---
id: B-drive-click-dead-after-gamepad-key
title: "After drive.key sends a gamepad key, drive.click (slate and os_input) no longer actuates CommonUI buttons and reports no_change_within_budget"
status: OPEN
severity: Medium
category: bug
tags: [drive, drive.click, drive.key, drive.hover, commonui, input-type, gamepad, os-input, no-change-within-budget, pie]
encounters: 1
lastSeen: 2026-09-24T10:23:00Z
---

# Clicks stop working once a gamepad key was injected

## Symptom

In a standalone PIE lesson, `drive.key {key:"Gamepad_Special_Right"}` (the game's UI.Action.Escape binding) opened
the in-game pause menu (log: `Applying input config for leaf-most node [W_HUD_DroneGameMenu_C_0]`). From then on
every `drive.click` on visible, enabled `SCommonButton`s (pause menu «Назад» / «Домой», and after closing the menu
by other means the lesson's own «ОК» buttons) returned `outcome:"no_change_within_budget"` and nothing happened,
both with the default Slate path and with `os_input:true`; `drive.hover os_input:true` first did not help. The
same buttons clicked fine earlier in the session and again after `editor.stop` + `editor.play`. The likely cause is
CommonUI's input type switching to Gamepad on the injected gamepad key, after which injected mouse presses are
not treated as clicks; nothing in the response says so.

## Repro

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv`, plugin `ba115afb`. `editor.play`; any CommonUI screen;
`drive.key {key:"Gamepad_Special_Right"}` (or any gamepad key); `drive.click` a visible `SCommonButton`.

## Ask

Either make `drive.click`/`drive.hover` move CommonUI back to mouse input (a real mouse move does), or report the
current CommonUI input type in the action response when a click lands on a button that ignores it.

## History

- `#1-gamepad-key-kills-clicks` `OPEN` reporter - Filed from a school lesson-attempt PIE session (plugin
  `ba115afb`); workaround was `editor.stop` + `editor.play`, or calling the target's handler via
  `object.call_function`.
