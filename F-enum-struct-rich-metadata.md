---
id: F-enum-struct-rich-metadata
title: "UEnum entry display-name/tooltip + UStruct field defaults/metadata via RPC"
status: DONE
severity: Low
category: feature
tags: [blueprint, user-defined-enum, user-defined-struct, dump-parity, metadata]
---

# UEnum entry display-name/tooltip + UStruct field defaults/metadata via RPC

The write surface for user-defined enums and structs is currently bare names + types only, while the read surface (post `F-rpc-blueprint-describe-struct`) returns rich per-field metadata. This violates dump/list parity from the authoring side: an agent can read displayName/tooltip/defaultValue/metaData but cannot set them.

Current state (`Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/BlueprintTypeDefinitionHandler.cpp`):

- `blueprint.set_enum_entries` (~line 632): `entries` param is an array of strings. No way to set per-entry display name, tooltip, or hidden flag. `ApplyEnumEntries` calls only the rename/insert path.
- `blueprint.add_struct_field` (~line 786): takes only `path`, `fieldName`, `fieldType`. No default value, no tooltip, no metadata, no per-field flags (`dontEditOnInstance`, `enableSaveGame`, `multiLineText`, `enable3dWidget`).
- `blueprint.create_struct` seed loop (~line 731) uses the same impoverished `{name,type}` shape.

**Proposed extension (additive, backward compatible):**

1. `blueprint.set_enum_entries` — accept either array-of-strings (existing) or array-of-objects `{ name, displayName?, tooltip?, hidden? }`. Detect element shape per-entry. Apply display name via `FEnumEditorUtils::SetEnumeratorDisplayName` (writes `FUserDefinedEnumEditorData::DisplayNameMap`); apply tooltip via the standard `ToolTip` metadata key on the enumerator; apply hidden via the `Hidden` metadata key. The editor's UEnum editor reads from these same hooks, so it stays in sync.

2. `blueprint.add_struct_field` — accept optional `defaultValue`, `tooltip`, `metaData` (object), and `flags` (`{dontEditOnInstance, enableSaveGame, multiLineText, enable3dWidget}`). Use `FStructureEditorUtils::ChangeVariableTooltip`, `FStructureEditorUtils::ChangeVariableDefaultValue`, `FStructureEditorUtils::ChangeVariableMetaData`, and the matching boolean flag setters. Mirror the same optionals in `blueprint.create_struct`'s seed loop.

3. New `blueprint.set_struct_field_default(path, fieldName, value)` and `blueprint.set_struct_field_metadata(path, fieldName, key, value)` for post-add edits without re-adding the field.

Localized display names should follow the readable NSLOCTEXT pattern from CLAUDE memory (namespace and key both human-readable strings, never hex GUIDs).

**Verification:** widen `TestBlueprintHandlers.cpp` to round-trip a seeded enum/struct through `set_enum_entries` + `add_struct_field` with the new fields and assert the rich-schema read side (`blueprint.list_struct_fields`, `UserDefinedStructDumpBuilder` output) reflects every value.

## History
- `#1-initial-proposal` `OPEN` reporter — `blueprint.set_enum_entries` only accepts string arrays; `blueprint.add_struct_field` only takes name+type. Read side (post F-rpc-blueprint-describe-struct) already returns displayName, tooltip, defaultValue, metaData, flags — write side has no parity. Proposed: object-shape enum entries with `displayName`/`tooltip`/`hidden`; struct field optional `defaultValue`/`tooltip`/`metaData`/`flags` on add and create-seed; new `set_struct_field_default` and `set_struct_field_metadata` RPCs. Use `FEnumEditorUtils::SetEnumeratorDisplayName` and `FStructureEditorUtils::Change*` so editor UIs stay in sync.
- `#2-rich-enum-struct-metadata` `IN-REVIEW` developer — Extended `blueprint.set_enum_entries` and `blueprint.create_enum` to accept object-shape entries `{name, displayName?, tooltip?, hidden?}` (string-only form still works), applying displayName via FEnumEditorUtils::SetEnumeratorDisplayName and tooltip/hidden via UEnum::SetMetaData/RemoveMetaData on the enumerator index. Extended `blueprint.add_struct_field` and `blueprint.create_struct` field seeds to accept optional `defaultValue`, `tooltip`, `metaData`, `flags{dontEditOnInstance,enableSaveGame,multiLineText,enable3dWidget}` applied through FStructureEditorUtils::{ChangeVariableTooltip, ChangeVariableDefaultValue, ChangeEditableOnBPInstance, ChangeSaveGameEnabled, ChangeMultiLineTextEnabled (gated by CanEnableMultiLineText), Change3dWidgetEnabled (gated by CanEnable3dWidget), SetMetaData}. Added new RPCs `blueprint.set_struct_field_default` and `blueprint.set_struct_field_metadata` for post-add edits. Ticket's reference to FStructureEditorUtils::ChangeVariableMetaData was incorrect — used SetMetaData instead. Tests appended to TestBlueprintHandlers.cpp; counterfactual: removing object-shape parsing makes rich enum entries lose displayName/tooltip/hidden.
- `#3-verify-rich-metadata-roundtrip` `DONE` tester — Verified: schemas for `blueprint.set_enum_entries`/`add_struct_field`/`set_struct_field_default`/`set_struct_field_metadata` expose the new params. Created temp `/Game/App/UI/Test/E_McpVerifyTemp_F_enum_struct` via `blueprint.create_enum` with object-shape entries (Alpha+Beta, displayName/tooltip/hidden) and temp `/Game/App/UI/Test/S_McpVerifyTemp_F_enum_struct` via `blueprint.create_struct` with field Health{defaultValue:"42.5", tooltip, metaData:{UIMin:0,UIMax:100}, flags:{dontEditOnInstance,enableSaveGame}}; `blueprint.list_struct_fields` round-tripped every value exactly (defaultValue="42.500000", tooltip="Health field tip", flags+metaData preserved); enum `asset.dump` properties.json shows `DisplayNameMap:{Alpha:"Alpha Display", Beta:"Beta Display"}`. Temp assets deleted via `asset.delete`.
