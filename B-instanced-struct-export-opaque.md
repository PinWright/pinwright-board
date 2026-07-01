---
id: B-instanced-struct-export-opaque
title: "FInstancedStruct properties serialize as opaque {} across property.get / asset.dump / inspect_object (type-erased payload never read back)"
status: IN-REVIEW
severity: High
category: bug
tags: [properties, asset-dump, inspect-object, instanced-struct, chooser, serialization, read-back]
---

# FInstancedStruct serializes as opaque `{}` — its type-erased payload is never read back

Any `FInstancedStruct` UPROPERTY (single-valued or inside a `TArray<FInstancedStruct>`) is emitted as an empty JSON object `{}` by every read-back surface: `property.get`, `asset.dump` (`properties.json`), and `system.inspect.inspect_object`. The struct's actual contents — the inner `UScriptStruct` type it currently holds, and that struct's field values — are completely lost. The reader reports the property *exists* and is overridden, but its payload is invisible, so a caller cannot verify what was authored.

This bites hardest on **Chooser tables** (`UChooserTable`), whose entire authored state lives in `FInstancedStruct` arrays: `ColumnsStructs` (the column kinds + property bindings + cell predicate bands), `ResultsStructs` (the per-row result, e.g. which `StaticMesh` each row resolves to), `ContextData`, and the single-valued `FallbackResult`. After building a chooser end-to-end with `chooser.*`, none of the per-row mesh-to-index pairing, no column kind/binding, and no cell predicate value can be independently read back — the only surface that even hints the result meshes persisted is `asset.references` / `asset.get_dependencies_classified` (an aggregate dependency list, not the per-row mapping), and that only after a forced `editor.save_all`. `FInstancedStruct` is also the storage for StateTree nodes, PoseSearch entries, and other type-erased "instanced" data, so this is not chooser-specific.

This is the same defect *class* as the now-DONE struct-serialization bugs
([B-properties-struct-export-text-fallback](B-properties-struct-export-text-fallback.md) — single fixed `FStructProperty`;
[B-asset-dump-instanced-subobjects-not-recursed](B-asset-dump-instanced-subobjects-not-recursed.md) — Instanced *UObject* sub-properties),
but a distinct, still-open case: `FInstancedStruct` is a **type-erased** struct whose payload is NOT a reflected member, so the existing reflection-walk fixes do not cover it.

## Root cause

`Utils/PropertyExport.cpp` `StructToJsonObject(UStruct* Struct, const void* StructPtr)` walks `TFieldIterator<FProperty>(Struct, IncludeSuper)`. For an `FInstancedStruct` the `Struct` is `FInstancedStruct::StaticStruct()`, which has **no reflected UPROPERTYs** — the payload (a `UScriptStruct*` plus a heap-allocated memory block) is stored in non-reflected C++ members. The field iterator yields nothing, so the result is `{}`. There is no special case that reads `FInstancedStruct::GetScriptStruct()` + `GetMemory()` and recurses `StructToJsonObject(ScriptStruct, Memory)` on the inner type.

## What it should do

When a `FStructProperty` (or array element) is an `FInstancedStruct` (`SP->Struct == FInstancedStruct::StaticStruct()`, or the `FStructUtils`/`TBaseStructure` family), read the wrapper's current inner type via `GetScriptStruct()` and, if non-null, recurse `StructToJsonObject(ScriptStruct, GetMemory())` and emit the inner struct's fields — ideally tagged with the inner struct type name (e.g. `{"_struct":"FChooserStructResult","Asset":"/Game/.../SM_Lightbulb"}`) so the caller can tell which column/result kind a cell holds. An empty `FInstancedStruct` (null inner type) can stay `{}` or carry `{"_struct":null}`.

Apply the fix in the shared `StructToJsonObject` walker so all three read surfaces (`property.get`, `asset.dump`, `inspect_object`) inherit it at once. Add a regression test on a chooser fixture (or a minimal `FInstancedStruct` holder) asserting the inner struct fields appear after authoring.

## Verbatim repro

Asset built end-to-end via `chooser.*` and saved: `/Game/DemoRoom/Choosers/CH_DisplayPropSelector` (2 columns, 3 rows, `OutputObjectType=/Script/Engine.StaticMesh`; rows assigned `SM_Slider_Button` / `S_MathHall_DisplayButton_01` / `SM_Lightbulb`).

- `property.get { objectPath:"/Game/DemoRoom/Choosers/CH_DisplayPropSelector.CH_DisplayPropSelector", propertyName:"ResultsStructs" }`
  → `{"propertyName":"ResultsStructs","value":[{},{},{}], ... "assetClass":"ChooserTable"}` (three opaque rows — no mesh assignment visible).
- `property.get { ... propertyName:"ColumnsStructs" }` → `value:[{},{}]` (both columns opaque).
- `property.get { ... propertyName:"ResultsStructs[0].Asset" }`
  → `[PROPERTY_NOT_FOUND] Failed to resolve nested property path 'ResultsStructs[0].Asset': Property 'ResultsStructs[0]' not found in scope 'ChooserTable' (segment 1 of 2)` (cannot drill into the element either).
- `asset.dump` → `properties.json` shows `"ColumnsStructs": { "type":"TArray", "value":[{},{}] }`, `"ResultsStructs": { ... "value":[{},{},{}] }`, `"ContextData": { ... "value":[{}] }` — every `FInstancedStruct` element empty; sibling non-instanced props (`DisabledRows:[false,false,false]`, `OutputObjectType:"/Script/Engine.StaticMesh"`) serialize fine.
- `system.inspect.inspect_object` → `"FallbackResult":{"type":"FInstancedStruct","value":{}}` — single-valued case: the type is correctly identified as `FInstancedStruct` but its `value` is `{}`; arrays likewise `"ColumnsStructs":{"value":[{},{}]}`, `"ResultsStructs":{"value":[{},{},{}]}`, `"ContextData":{"value":[{}]}`.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed on `/Game/DemoRoom/Choosers/CH_DisplayPropSelector` (a saved `UChooserTable` authored via `chooser.*`). `property.get` returns `ResultsStructs=[{},{},{}]` and `ColumnsStructs=[{},{}]`; the nested path `ResultsStructs[0].Asset` is rejected `[PROPERTY_NOT_FOUND]`; `asset.dump`/`properties.json` and `inspect_object` both emit every `FInstancedStruct` element as `{}` (including the single-valued `FallbackResult:{"type":"FInstancedStruct","value":{}}`). Root cause: `Utils/PropertyExport.cpp` `StructToJsonObject` walks `TFieldIterator<FProperty>` over `FInstancedStruct::StaticStruct()`, which has no reflected payload UPROPERTYs, so the type-erased inner struct (`GetScriptStruct()`/`GetMemory()`) is never recursed. Same defect class as the DONE fixes B-properties-struct-export-text-fallback (fixed `FStructProperty`) and B-asset-dump-instanced-subobjects-not-recursed (Instanced UObject), but distinct: `FInstancedStruct` is type-erased and uncovered by either. Per-row mesh-to-index pairing and cell predicate values are unreadable on every read surface as a result.
- `#2-fix` `IN-REVIEW` developer — Fixed at the shared locus inside `StructToJsonObject` (`Source/EditorAutomationRpcGateway/Private/Utils/PropertyExport.cpp`) so all three read surfaces (`property.get`, `asset.dump`, `inspect_object`) inherit it at once: added a `__has_include` compat guard for `StructUtils/InstancedStruct.h` (5.5+) / `InstancedStruct.h` (≤5.4), then at the top of `StructToJsonObject` detect `Struct == FInstancedStruct::StaticStruct()`, read the wrapper's current inner type via `GetScriptStruct()` + `GetMemory()`, and recurse `StructToJsonObject(InnerType, Memory)` to surface the inner struct's reflected fields. The inner type is tagged with the file's established `_kind` marker convention (e.g. `{"_kind":"FChooserStructResult","Asset":...}`) rather than a new `_struct` key; an empty wrapper (null inner type) emits `{"_kind":null}`. Because this fix lands at the shared walker, both the single-valued `FStructProperty` branch and the `TArray<FInstancedStruct>` array-inner branch are covered with no per-call-site edit. Bumped the `properties.json` asset-dump aspect version 4→5 in `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetDumpCache.cpp` since serialized bytes change. Regression test added: `Source/EditorAutomationRpcGateway/Private/Tests/Utility/TestPropertyUtilsInstancedStruct.{h,cpp}` (`EditorAutomationRpcGateway.utils.property_utils.InstancedStructExport`) — a host UObject with a single `FInstancedStruct` and a `TArray<FInstancedStruct>`, each holding a known reflected payload, exported through the production `ExportPropertyToJsonValue`; asserts the inner `IntField`/`StringField`/`BoolField` and the `_kind` type name surface (would fail with `{}` if reverted), plus the empty-wrapper `_kind` path. Note: the `#1` nested-path footnote `ResultsStructs[0].Asset → [PROPERTY_NOT_FOUND]` is a separate `UtilityPropertyHandler` path-resolver gap and is intentionally out of scope here; this fix makes the element contents visible in the array/single-value dumps. Not compiled/tested in this phase (a later phase does).
