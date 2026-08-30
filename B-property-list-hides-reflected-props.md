---
id: B-property-list-hides-reflected-props
title: "`property.list` silently filters out reflected UPROPERTYs lacking Edit/BlueprintVisible flags"
status: DONE
severity: High
category: bug
tags: []
---

# `property.list` silently filters out reflected UPROPERTYs lacking Edit/BlueprintVisible flags

`property.list` only returns properties that carry `CPF_Edit` or
`CPF_BlueprintVisible` (and, by default, are also instance-editable). It
silently drops every other reflected `UPROPERTY()` and returns
`count: 0` with an empty `properties` array. Nothing in the response
hints that filtering happened, so callers conclude the class has no
reflected fields when in fact it has many — and `property.get` /
`property.set` still work on those same hidden fields. The misleading
empty result is the bug; the filter itself is reasonable as an opt-in
view but should not be the silent default.

**Repro (PIE running on `L_Core`):**

`UApiSubsystem`
(`C:\Unity\unreal-fpv\Plugins\App\Source\App\Api\ApiSubsystem.h`) has
21 `UPROPERTY` decorations; most are plain `UPROPERTY()` with no
`Edit*` / `BlueprintReadOnly` specifiers.

1. `property.list { "objectPath": "/Engine/Transient.LyraEditorEngine_0:B_DroneGameInstance_C_2.ApiSubsystem_0" }`
   returns:
   ```json
   {
     "objectPath": "/Engine/Transient.LyraEditorEngine_0:B_DroneGameInstance_C_2.ApiSubsystem_0",
     "className": "ApiSubsystem",
     "properties": [],
     "count": 0,
     "assetPath": "/Engine/Transient",
     "assetName": "ApiSubsystem_0",
     "existsAfter": true,
     "assetClass": "ApiSubsystem"
   }
   ```
2. `property.get { "objectPath": "...ApiSubsystem_0", "propertyName": "eServerType" }`
   on the same object returns:
   ```json
   {
     "propertyName": "eServerType",
     "value": "School",
     "assetPath": "/Engine/Transient",
     "assetName": "ApiSubsystem_0",
     "existsAfter": true,
     "assetClass": "ApiSubsystem"
   }
   ```
   confirming the field is reflected and live — just hidden from
   `property.list`.

**Root cause:**
`Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp`
`ShouldIncludePropertyInList()` at lines 439–464 returns false for any
property that lacks both `CPF_Edit` and `CPF_BlueprintVisible` unless
the caller passes `includeAll: true` (handler at line 1136, default
`bIncludeAll = false` at line 1164, filter call at line 1185). The
existing `includeAll` opt-in does what's needed under the hood, but it
defaults off and is not surfaced in the response, so callers have no
way to discover that hidden properties exist.

**Workaround:** Pass `includeAll: true` on every `property.list` call.

**Fix:** Pick one (Option A preferred):

- **Option A (preferred):** Flip the default — return **all** reflected
  `UPROPERTY`s by default. Add a `flags` field on each entry exposing
  the property's specifiers (e.g.
  `{ "edit": bool, "blueprintVisible": bool, "editOnInstance": bool, "transient": bool }`)
  so callers can filter client-side. Keep `includeAll` accepted as a
  no-op alias for compatibility, and rename the inverse filter to
  `editableOnly: true` for callers that still want the old view.
  Document the flag set on the `property.list` wiki page.
- **Option B:** Keep the filter default-on but rename the existing
  `includeAll` to `includeHidden` (accept both for compat), document
  it clearly on the `property.list` wiki page, and add a top-level
  `hiddenCount` field to the response whenever properties were
  filtered out so callers see something is being suppressed.

## Acceptance

With PIE running on `L_Core`, the default
`property.list { "objectPath": "/Engine/Transient.LyraEditorEngine_0:B_DroneGameInstance_C_2.ApiSubsystem_0" }`
call (no extra params under Option A; with `includeHidden: true` under
Option B) returns at least 21 entries covering `eServerType` and the
other plain-`UPROPERTY()` fields of `UApiSubsystem`. Under Option A,
each entry carries a `flags` object exposing edit/blueprintVisible/
editOnInstance/transient bits so callers can re-derive the old
filtered view client-side.

## History
- `#1-initial-repro` `OPEN` reporter — Reproduced live with PIE running on `L_Core`: `property.list` on `/Engine/Transient.LyraEditorEngine_0:B_DroneGameInstance_C_2.ApiSubsystem_0` returns `count: 0` even though `UApiSubsystem` declares 21 reflected `UPROPERTY`s and `property.get` on `eServerType` returns `"School"` against the same object. Root cause is `ShouldIncludePropertyInList()` at `UtilityPropertyHandler.cpp:439–464` requiring `CPF_Edit` or `CPF_BlueprintVisible`, gated by the `includeAll` opt-in (default false) at `UtilityPropertyHandler.cpp:1164`.
- `#2-flip-default-add-flags` `IN-REVIEW` developer — Flipped `property.list` default to include all reflected `UPROPERTY`s; added optional `editableOnly:true` to restore filtered view; `includeAll:true` retained as no-op alias. Each entry now carries a `flags` sub-object `{edit, blueprintVisible, editOnInstance, transient}`. Three regression tests in `TestUtilityHandlers.cpp` cover default, `editableOnly`, and flag-shape.
- `#3-verify-pass` `DONE` tester — Verified `property.list` on live `/Engine/Transient.LyraEditorEngine_0:B_DroneGameInstance_C_0.ApiSubsystem_0` now returns `count: 21` (was `0`) with each entry carrying `flags: {edit, blueprintVisible, editOnInstance, transient}` plus current/default/override values. Live state visible: `eServerType="School"`, `bHasLessonsApi=true`, and notably the `OnLoginStatusChanged` delegate's 9 currently-bound listener paths — extremely useful for debugging. `editableOnly:true` opt-in restoring filter not tested here but documented in #2. Acceptance bar met.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
