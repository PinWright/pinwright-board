---
id: E-asset-dump-instanced-uobject-class-elided
title: "asset.dump emits Instanced UObject sub-objects with no class discriminator — a CDO-identical polymorphic element is an indistinguishable `{}`"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [asset-dump, properties, instanced-subobjects, smart-object, behavior-definition, class-discriminator, read-back]
---

# Instanced UObject sub-objects dump without their class name — `{}` hides which subclass is attached

`asset.dump` recurses into Instanced UObject sub-properties as a **sparse** field
diff against the subobject's archetype/CDO (correct, per
[B-instanced-subobject-no-cdo-diff](B-instanced-subobject-no-cdo-diff.md)). But
the sparse object it emits carries **no class/type discriminator** for the
subobject. When a polymorphic instanced element is left at its class defaults
(zero overridden fields), it serializes as a bare `{}` — indistinguishable from
an empty/absent element and giving no hint of *which* concrete subclass is
attached.

This bites hardest on `TArray<USmartObjectBehaviorDefinition>` (a slot's
`BehaviorDefinitions`), where the concrete subclass IS the load-bearing payload
(the whole point of the array is *which behavior*), but the same applies to any
polymorphic `Instanced` UObject array. After authoring a slot behavior via
`ai.configure_slot_behavior` (which now correctly attaches the behavior and even
returns `behaviorClass` in its own response), the `asset.dump` readback can
confirm only the array *length* (1 = configured vs 0 = default), not the class —
so "confirm the slot behaviors stuck" cannot be satisfied from the dump alone.

The sibling type-erased case already solved this: the `FInstancedStruct` fix
([B-instanced-struct-export-opaque](B-instanced-struct-export-opaque.md),
IN-REVIEW) tags the inner payload with a `_kind` marker
(`{"_kind":"FChooserStructResult","Asset":...}`) precisely so the caller can tell
which kind a cell holds. The Instanced-UObject branch should follow the same
convention.

## Root cause (`Utils/PropertyExport.cpp`)

`ExpandInstancedSubobject` → `SparseInstancedSubobjectToJsonObject` (~line 268)
returns `BuildSparseFieldDiffJson(Subobject, ResolveInstancedSubobjectBaseline(...), ...)`
— purely the sparse field diff. It never adds `Subobject->GetClass()->GetPathName()`
(or a `_kind`/`_class` key), so a CDO-identical instanced UObject collapses to
`{}` with the concrete class erased.

## Verbatim demonstration (live editor, `mcp__editor-automation__call`)

Definition `/Game/AI/SmartObjects/SOD_CoffeeStation`, slot 0 configured with
`ai.configure_slot_behavior {slotIndex:0, behaviorType:"/Script/MassSmartObjects.SmartObjectMassBehaviorDefinition", ...}`
(the configure call itself returned `behaviorClass:"/Script/MassSmartObjects.SmartObjectMassBehaviorDefinition"`).

`asset.dump {assetPath:"/Game/AI/SmartObjects/SOD_CoffeeStation"}` →
`properties.json`, slot 0:

```json
"BehaviorDefinitions": [
    {
    }
],
```

The attached `SmartObjectMassBehaviorDefinition` is a bare `{}` — no class name.
Slot 1 (default, no behavior) dumps `"BehaviorDefinitions": []`. So the only
readable distinction is length 1 vs 0; the concrete behavior subclass is invisible
in the dump, and a configured-at-defaults element is indistinguishable from `{}`.

## What it should do

When `ExpandInstancedSubobject` serializes an Instanced UObject sub-object, add a
type discriminator (e.g. `{"_kind":"<Subobject->GetClass()->GetPathName()>", ...sparse fields...}`),
matching the `_kind` convention already used for the `FInstancedStruct` branch.
A defaults-only element then reads `{"_kind":"/Script/MassSmartObjects.SmartObjectMassBehaviorDefinition"}`
instead of `{}`, making polymorphic instanced arrays readable. Bump the
`properties.json` aspect version (serialized bytes change).

## History
- `#1-initial-repro` `OPEN` reporter — Seed-mode fuzz on `ai.configure_slot_behavior` (coffee-station smart object task); the seed method itself now works correctly (behavior attached, bad behaviorType rejected with `availableBehaviorTypes`, tags applied — its prior no-op bug is the IN-REVIEW B-configure-slot-behavior-ignores-behavior-and-tags). The residual friction is in the `asset.dump` readback: replay-confirmed live on `/Game/AI/SmartObjects/SOD_CoffeeStation`, slot 0's attached `SmartObjectMassBehaviorDefinition` dumps as `"BehaviorDefinitions":[{}]` (bare `{}`, no class), slot 1 as `[]` — so the readback can confirm array length but not which behavior subclass is attached, leaving "confirm the slot behaviors stuck" unsatisfiable from the dump. Root cause: `Utils/PropertyExport.cpp` `SparseInstancedSubobjectToJsonObject`/`ExpandInstancedSubobject` (~268) emit only the sparse field diff and never tag the subobject class, unlike the sibling `FInstancedStruct` branch which adds a `_kind` marker (B-instanced-struct-export-opaque). Distinct from the DONE instanced-UObject tickets (recursion + sparse-diff), which addressed *field* visibility, not the missing class discriminator. Culprit method: `asset.dump`.
- `#2-class-discriminator-tag` `IN-REVIEW` developer — Fixed. `ExpandInstancedSubobject` (`Utils/PropertyExport.cpp`) now tags every serialized Instanced UObject sub-object with `_kind` = `Subobject->GetClass()->GetPathName()` after building the sparse field diff, matching the `_kind` convention the FInstancedStruct branch (`StructToJsonObject`) already uses. A defaults-only / CDO-identical polymorphic element now reads `{"_kind":"/Script/MassSmartObjects.SmartObjectMassBehaviorDefinition"}` instead of a bare `{}`, so the concrete subclass attached to e.g. a `BehaviorDefinitions` slot is readable from the dump. The path name matches the class identity the configure / write-time RPCs echo. Bumped `properties.json` aspect version 5→6 in `Handlers/Asset/AssetDumpCache.cpp` (serialized bytes change) per the Aspect Version Bumping rule. Regression test: added `FPropertyUtilsInstancedSubobjectClassDiscriminatorTest` (`EditorAutomationRpcGateway.utils.property_utils.InstancedSubobjectClassDiscriminator`) to `Private/Tests/Utility/TestAssetDumpInstancedSubobjects.cpp` — exercises production `ExportPropertyToJsonValue` on a top-level instanced subobject and on a defaults-only array element, asserting both carry `_kind` = the subobject class path and that the defaults-only element is no longer an empty object; it fails if the `_kind` tag is reverted. Files: `Source/EditorAutomationRpcGateway/Private/Utils/PropertyExport.cpp`, `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetDumpCache.cpp`, `Source/EditorAutomationRpcGateway/Private/Tests/Utility/TestAssetDumpInstancedSubobjects.cpp`.
