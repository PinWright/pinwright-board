---
id: F-enum-append-entry-keep-names
title: "No RPC can append an enumerator to an existing user-defined enum: blueprint.set_enum_entries rejects the enum's own existing names, so keeping NewEnumerator0..N intact is impossible"
status: OPEN
severity: Medium
category: feature
tags: [blueprint, user-defined-enum, set-enum-entries, enum-editor-utils, dependents-refresh]
encounters: 1
lastSeen: 2026-09-23T18:30:00Z
---

# No RPC can append an enumerator to an existing user-defined enum

Task: add one value to `/App/App/UI/LobbyAndMenu/Popups/E_LoginOverlayState` (7 entries, internal names `NewEnumerator0..6`, used by enum-indexed `K2Node_Select` nodes and `Set State` calls in two widgets). Every existing reference is keyed by those internal names, so they must not change.

`blueprint.set_enum_entries` with all eight entries (the seven existing `NewEnumeratorN` names plus `NewEnumerator7`, each with a `displayName`) fails with `ENUM_UPDATE_FAILED: Enum entry 'NewEnumerator0' is not a valid user-defined enum enumerator name`. `ValidateAndSanitizeEnumEntries` runs `FEnumEditorUtils::IsProperNameForUserDefinedEnumerator` against the enum being edited, which rejects any name the enum already contains, so the verb can only rename every entry and break every pin that references the enum. It also writes through `UEnum::SetEnums` directly rather than `FEnumEditorUtils::AddNewEnumeratorForUserDefinedEnum` / `BroadcastChanges`, so dependent Blueprints' enum-indexed Select nodes would not be refreshed.

**Workaround used:** `editor.open_asset` on the enum, `drive.click` on the enum editor's "Add Enumerator" button (`surface: editor_chrome`), then `drive.type` into the new row's display-name field. The engine path refreshed the dependent Select nodes (a `NewEnumerator7` pin appeared).

**Fix:** a `blueprint.add_enum_entry` (append via `FEnumEditorUtils::AddNewEnumeratorForUserDefinedEnum` + `SetEnumeratorDisplayName`), or let `set_enum_entries` keep names the enum already has at the same index and route through `FEnumEditorUtils` so dependents are notified.

**Source (8748c637):** `BlueprintTypeDefinitionHandler.cpp:370` runs `IsProperNameForUserDefinedEnumerator` against the enum being edited; `:420-432` rewrites the list with `UEnum::SetEnums`; `:493-494` ends with `MarkPackageDirty` + save and no `FEnumEditorUtils` change broadcast.

## History
- `#1-append-impossible-without-rename` `OPEN` reporter — UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`, school-computer login work. The editor-chrome workaround cost about ten extra calls.
