---
id: B-drive-type-pie-editable-text-no-input
title: "drive.type into a PIE EditableText does not land (focus goes to an editor SDockingTabStack); drive.key ctrl+A does not select-all in editor chrome"
status: OPEN
severity: Medium
category: bug
tags: [drive, drive.type, drive.key, pie, editable-text, focus, keyboard, editor-chrome]
encounters: 1
lastSeen: 2026-09-23T18:30:00Z
---

# drive.type does not reach a PIE text box

PIE in the level-editor viewport, popup `W_ChangeServer` pushed on `UI.Layer.Menu`. `drive.type {handle: W_OverallUILayout_C_0/W_ChangeServer_C_0/SEditableText, text: ...}` returns `no_change_within_budget`; the widget's `NameTextBox` stays empty (the popup then reports "Server URL is empty"). After a `drive.click` on the same handle, `drive.input_state.focused_widget` is `SDockingTabStack [TabManager.cpp(2122)]`, not the text box. `drive.type` into editor-chrome text fields does work (enum editor), but `drive.key {key: A, modifiers: ctrl}` there did not select the existing text, so typed text was appended.

**Workaround:** PIE-only Python `set_text` on the live EditableTextBox; in editor chrome, 40x BackSpace and 40x Delete before typing.

**Source (8748c637):** `drive.type` clicks the element center, then types (`DriveActionHandlers.cpp:257-258`); `FDriveInput::TypeString` sends KeyDown/Char/KeyUp to whatever Slate user focus holds (`DriveInput.cpp:391-439`), so text is lost whenever the focusing click does not move keyboard focus into the text box. `drive.key` builds modifiers into `FModifierKeysState` on the key event only (`DriveInput.cpp:352-389`).

**Related:** `B-drive-click-misses-pie-game-viewport` (OPEN), same PIE-in-level-viewport configuration; the focus landing on `SDockingTabStack` is consistent with its `#3` capture/focus findings.

## History
- `#1-type-lost-in-pie` `OPEN` reporter — UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`.
