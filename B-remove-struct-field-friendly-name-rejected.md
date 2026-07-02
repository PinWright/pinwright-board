---
id: B-remove-struct-field-friendly-name-rejected
title: "blueprint.remove_struct_field rejects a field's friendly/displayName (only matches mangled internal name), unlike its sibling field-edit RPCs"
status: IN-REVIEW
severity: Medium
category: bug
tags: [blueprint, user-defined-struct, remove_struct_field, friendly-name, internal-name, sibling-inconsistency]
---

# `blueprint.remove_struct_field` rejects the field's friendly name; sibling struct-field RPCs accept it

Within the user-defined-struct field family the methods disagree on what
`fieldName` may be, so the obvious round-trip breaks:

- `blueprint.add_struct_field {fieldName:"MaxStack"}` — the `fieldName` you pass
  becomes the field's **friendly name**. `list_struct_fields` then reports that
  field as `{"name":"MaxStack_4_<GUID>", "displayName":"MaxStack", ...}` — the
  `name` is a mangled internal name, `displayName` is what you typed.
- `blueprint.set_struct_field_default {fieldName:"MaxStack"}` — **works** by
  friendly name (uses `FindStructFieldGuidByName`, which matches `FriendlyName`
  OR `VarName`). Its wiki/source param doc reads "Friendly or internal name of
  the field".
- `blueprint.set_struct_field_metadata {fieldName:"MaxStack"}` — **works** by
  friendly name, same helper, same "Friendly or internal name" doc.
- `blueprint.remove_struct_field {fieldName:"MaxStack"}` — **FAILS** with
  `[FIELD_NOT_FOUND] Field 'MaxStack' was not found`. It only matches
  `VarDesc.VarName` (the mangled internal name), never `FriendlyName`. To remove
  the field you must first `list_struct_fields`, read the mangled
  `name` (`MaxStack_4_<GUID>`), and pass that.

So a caller who created a field by friendly name, or who read `displayName` from
`list_struct_fields`, and tries to delete it by that same name is wrongly
rejected — even though two sibling field-edit RPCs accept exactly that name, and
even though the friendly name is the only human-meaningful identifier (the
internal name is GUID-mangled and never user-supplied). This is an asymmetric
resolution bug, not just a doc gap: the displayName is a valid, documented field
identifier everywhere else in the family.

The source comment is also wrong about the current behavior: the helper at
`BlueprintTypeDefinitionHandler.cpp` reads *"Friendly-name-first, then
internal-name lookup mirroring remove_struct_field"* — but `remove_struct_field`
does **not** do friendly-name lookup, so they do not mirror.

## What it should do

`blueprint.remove_struct_field` should resolve `fieldName` the same way its
siblings do — match `VarDesc.FriendlyName` OR `VarDesc.VarName` (case-insensitive),
i.e. call the existing `FindStructFieldGuidByName` helper instead of the inline
`VarName`-only loop (`BlueprintTypeDefinitionHandler.cpp`, the
`blueprint.remove_struct_field` handler, the field-match loop iterating
`FStructureEditorUtils::GetVarDesc`). Then update the wiki/param doc to
"Friendly or internal name" to match the siblings. The `FindStructFieldGuidByName`
comment about "mirroring remove_struct_field" becomes true once removal also
checks FriendlyName.

**Workaround:** `blueprint.list_struct_fields` first, then pass the mangled
internal `name` (e.g. `MaxStack_4_<GUID>`) — not the `displayName` — to
`remove_struct_field`.

## Evidence (live replay, this build)

Created `/Game/Data/S_OracleReplay` seeded with fields `ItemName` (string) and
`MaxStack` (int).

`blueprint.list_struct_fields {"path":"/Game/Data/S_OracleReplay"}` →
```json
{"fields":[
  {"name":"ItemName_2_B393663842AF433990E77EBF32FBD04E","displayName":"ItemName","type":"string", ...},
  {"name":"MaxStack_4_14205DBE4DC7C35BDF6F99BB4ADDB408","displayName":"MaxStack","type":"int", ...}],
 "count":2, ...}
```

`blueprint.remove_struct_field {"path":"/Game/Data/S_OracleReplay","fieldName":"MaxStack"}` →
```
[FIELD_NOT_FOUND] Field 'MaxStack' was not found
```

`blueprint.remove_struct_field {"path":"/Game/Data/S_OracleReplay","fieldName":"MaxStack_4_14205DBE4DC7C35BDF6F99BB4ADDB408"}` →
```json
{"success":true,"fieldName":"MaxStack_4_14205DBE4DC7C35BDF6F99BB4ADDB408", ...}
```

Same `displayName` ("MaxStack") is accepted by `set_struct_field_default` /
`set_struct_field_metadata` (helper checks `FriendlyName`) but rejected by
`remove_struct_field` (inline loop checks only `VarName`).

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed (seed `blueprint.list_struct_fields`; culprit `blueprint.remove_struct_field`). `add_struct_field {fieldName:"MaxStack"}` creates a field whose `displayName` is "MaxStack" and whose internal `name` is `MaxStack_4_<GUID>`. `remove_struct_field {fieldName:"MaxStack"}` returns `[FIELD_NOT_FOUND]`; the same friendly name works on `set_struct_field_default`/`set_struct_field_metadata` (which call `FindStructFieldGuidByName` matching `FriendlyName` OR `VarName`), and removal by the mangled internal name succeeds. Root cause: the `remove_struct_field` handler's inline match loop checks only `VarDesc.VarName`, never `FriendlyName`, so it diverges from the rest of the struct-field family. The `FindStructFieldGuidByName` comment claiming it "mirrors remove_struct_field" is stale/inaccurate. Fix: have `remove_struct_field` use `FindStructFieldGuidByName` (or add the `FriendlyName` branch) and update its param doc to "Friendly or internal name". Not a dup of `F-enum-struct-rich-metadata` (rich-metadata write parity, DONE) or `F-rpc-blueprint-describe-struct` (rich read parity, DONE); neither covers removal-by-friendly-name resolution.
- `#2-fix` `IN-REVIEW` developer — Routed `blueprint.remove_struct_field` through the shared `FindStructFieldGuidByName` helper (matches `FriendlyName` OR `VarName`, case-insensitive) instead of its inline `VarName`-only loop, so removal now accepts the field's friendly `displayName` exactly like `set_struct_field_default`/`set_struct_field_metadata`. To share the helper across all three handlers I moved its definition up into the earlier anonymous namespace (next to `DoesStructFieldNameExist`, which already does the identical `FriendlyName || VarName` match) and deleted the duplicate definition that previously sat just above `set_struct_field_default`; updated the helper's stale comment (it no longer claims to "mirror remove_struct_field" since removal now genuinely shares it) and bumped the `remove_struct_field` `fieldName` param doc from "Struct field name" to "Friendly or internal name of the field" to match the siblings. File: `Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/BlueprintTypeDefinitionHandler.cpp`. Regression test `FBlueprintRemoveStructFieldByFriendlyNameTest` (`EditorAutomationRpcGateway.blueprint.remove_struct_field.AcceptsFriendlyName`) added to `Source/EditorAutomationRpcGateway/Private/Tests/Blueprint/TestBlueprintHandlers.cpp`: seeds a struct field whose friendly name "Score" differs from its mangled internal name, asserts that divergence, then deletes by the friendly name through the real registered handler and asserts success + absence on read-back; reverting to the `VarName`-only loop makes the delete return `FIELD_NOT_FOUND` and the test fails on `Capture.bSuccess`.
