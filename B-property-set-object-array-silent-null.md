---
id: B-property-set-object-array-silent-null
title: "property.set silently stores null for object-array (TArray<UObject*>) elements it cannot resolve — returns applied:true with value [null]"
status: IN-REVIEW
severity: High
category: bug
tags: [property, property-set, property-import, object-array, instanced-subobject, silent-noop, success-no-effect]
encounters: 1
lastSeen: 2026-07-04T22:40:49.3952411+03:00
---

# `property.set` on an object array silently coerces unresolvable elements to null and reports success

Replacing a whole object-array UPROPERTY through `property.set` — an
`FArrayProperty` whose `Inner` is an `FObjectProperty` (e.g.
`UInputAction.Modifiers`, an Instanced `TArray<TObjectPtr<UInputModifier>>`) —
silently stores `null` for any array element the importer can't turn into a
loaded `UObject`, then returns `applied:true`. The array ends up populated with
bogus `null` entries and the caller gets a clean success.

The shared importer's array-inner object branch has **two** silent no-op paths,
both of which return success:

1. **A JSON-object element** (the natural "instanced subobject" shape, e.g.
   `{"$class":"InputModifierNegate","bX":false,"bY":true,"bZ":false}`): `V->Type`
   is `EJson::Object`, so `ObjPath` is set to the empty string, which
   `IsNullObjectSentinel("")` treats as the null sentinel — the load branch is
   skipped entirely (no load attempt, not even the warning), and the element is
   stored as `null`.
2. **A non-loadable string path**: only `UE_LOG(Warning)` fires (invisible to the
   RPC caller); the element is still stored as `null` and the call still succeeds.

Contrast the **scalar** `FObjectProperty` branch in the same importer
(`PropertyImport.cpp:665-706`), which correctly returns `false` — `"Failed to
load object at path: <p>"` for an unloadable path and `"Unsupported JSON type for
object property"` for a non-string/non-null value. The array-inner branch is
missing both guards, so an object *array* fails silently where a scalar object
property fails loud.

**Why it matters:** this is a silent success-with-no-effect on the generic
reflected writer. A caller that populates an object array via `property.set`
(input modifiers/triggers, AIPerception `SensesConfig`, component/instanced
arrays, etc.) reads back `applied:true`, believes the array was authored, and
builds on an array full of `null`. The readback (`property.get` on the same
property) returns `[null]` — a real element count wrapping nothing — which then
needs an out-of-band manual clear. There is no `applied:false`, no error, no
RPC-visible warning.

## Guilty source

`Plugins/PinWright/Source/PinWright/Private/Utils/PropertyImport.cpp:1006-1022`
(the `FArrayProperty` branch's `FObjectProperty` inner case):

```cpp
if (FObjectProperty* ObjInner = CastField<FObjectProperty>(Inner))
{
    FString ObjPath = (V->Type == EJson::String) ? V->AsString() : FString();
    UObject* LoadedObj = nullptr;
    // {"", "None", "null"} clear the element to null; only a real path loads.
    if (!IsNullObjectSentinel(ObjPath))
    {
        LoadedObj = LoadObject<UObject>(nullptr, *ObjPath);
        if (!LoadedObj) LoadedObj = StaticLoadObject(UObject::StaticClass(), nullptr, *ObjPath);
        if (!LoadedObj)
        {
            UE_LOG(LogTemp, Warning, TEXT("Failed to load object at path: %s (array element %d)"), *ObjPath, i);
        }
    }
    ObjInner->SetObjectPropertyValue(ElemPtr, LoadedObj);
    continue;
}
```

A JSON-object element makes `ObjPath` empty (line 1008) → `IsNullObjectSentinel`
true (line 1011) → the load body is skipped → `SetObjectPropertyValue(ElemPtr,
nullptr)` (line 1020). A non-loadable path only warns (lines 1015-1018) and still
stores null (line 1020). The outer `FArrayProperty` branch then returns `true`.

## What it should do / fix

Mirror the scalar `FObjectProperty` guards in the array-inner branch: reject a
non-string, non-null element (`return false`, e.g. `"Array element N: unsupported
JSON type for object property"`) instead of silently emptying the path; and
reject a non-sentinel path that fails to load (`return false`, not just
`UE_LOG`). That converts both silent drops into a real `applied:false` error the
caller can see.

Note the deeper capability gap this task actually hit: an Instanced / EditInline
object array (`UInputAction.Modifiers`) has no *existing* object to point a path
at — authoring one requires `NewObject`-ing the `{"$class":...}`-named subclass
and recursing its fields, which no importer branch does. That authoring gap is
tracked separately (`B-input-trigger-modifier-stub-silent-success` for the input
verbs). This ticket is scoped to the narrower, generic defect: `property.set`
must **fail loud**, not store a silent `null`, when it can't resolve an
object-array element.

## Repro (live, replay-confirmed at HEAD)

Target: `/Game/Input/IA_Look.IA_Look` (a `UInputAction`; `Modifiers` is its
Instanced `TArray<UInputModifier>`).

1. `property.set { objectPath:"/Game/Input/IA_Look.IA_Look", propertyName:"Modifiers", value:[{"$class":"InputModifierNegate","bX":false,"bY":true,"bZ":false}] }`
   -> `{"propertyName":"Modifiers","applied":true,"markedDirty":true,"assetPath":"/Game/Input/IA_Look","assetName":"IA_Look","existsAfter":true,"assetClass":"InputAction","value":[null]}`
   (reports `applied:true` but `value` is `[null]` — the modifier was never created)
2. readback — `property.get { objectPath:"/Game/Input/IA_Look.IA_Look", propertyName:"Modifiers" }`
   -> `{"propertyName":"Modifiers","value":[null], ...}` — the array holds a bogus
   `null`, no `InputModifierNegate` was authored.

## History
- `#2-fix-fail-loud` `IN-REVIEW` developer — Fixed the array-inner `FObjectProperty` branch to fail loud, mirroring the scalar `FObjectProperty` branch (PropertyImport.cpp:665-706). In `ApplyJsonValueToProperty`'s `FArrayProperty`→`FObjectProperty` inner case (PropertyImport.cpp:1006-1022): a non-string/non-null JSON element (e.g. a `{"$class":...}` object, a number, a bool) now returns `false` with `"Array element N: unsupported JSON type for object property"` instead of collapsing to an empty path that `IsNullObjectSentinel` treats as the null sentinel; and a non-sentinel string path that fails to load now returns `false` with `"Array element N: failed to load object at path: <p>"` instead of only `UE_LOG(Warning)` + storing null. The legitimate clear cases (JSON `null` and the `""`/`"None"`/`"null"` string sentinels) still store null and return `true`. This routes the reported repro (`property.set` on `IA_Look.Modifiers` with a `[{"$class":"InputModifierNegate",...}]` value) from `applied:true`+`value:[null]` to a loud `PROPERTY_CONVERSION_FAILED` error (UtilityPropertyHandler.cpp:1095-1099). Files: `Plugins/PinWright/Source/PinWright/Private/Utils/PropertyImport.cpp`. Added regression test `PinWright.utils.property_import.ObjectArrayElementFailLoud` (new `Private/Tests/Utility/TestObjectArrayImportFailLoud.cpp` + fixture `TestObjectArrayImportFixture.h`, a transient `TArray<TObjectPtr<UObject>>` host) which drives production `ApplyJsonValueToProperty` and asserts fail-loud on JSON-object / unloadable-path / number elements while preserving the null + `"None"` clear cases and a loadable-path happy path. Scope unchanged: the deeper instanced-subobject authoring gap stays tracked in `B-input-trigger-modifier-stub-silent-success`.
- `#1-initial-repro` `OPEN` reporter — Realism-mode Enhanced-Input task (author a Negate/invert-Y modifier on IA_Look). The intended verb `input.set_input_modifier` now returns `NOT_IMPLEMENTED` (its fail-loud fix from `B-input-trigger-modifier-stub-silent-success`, IN-REVIEW), so the attempt fell back to `property.set` on the `Modifiers` array — which returned `applied:true` with `value:[null]`. Replay-confirmed live: `property.set` on `/Game/Input/IA_Look.IA_Look` `Modifiers` with a `[{"$class":"InputModifierNegate",...}]` value returned `applied:true` and `property.get` read back `[null]` — no modifier created. Root-caused to `PropertyImport.cpp:1006-1022`: the `FArrayProperty` branch's `FObjectProperty` inner case maps a JSON-object element to an empty path (line 1008), which `IsNullObjectSentinel` treats as the null sentinel (line 1011), so it stores `null` (line 1020) with no load attempt and no error; an unloadable string path only `UE_LOG`s a warning (1015-1018) and also stores `null`. Both return success, unlike the scalar `FObjectProperty` branch (665-706) which errors. Distinct from `B-property-set-saved-true-not-persisted` (the `saved` field), `B-property-path-silent-world-fallback` (object resolver climbs to World), and `B-container-array-append-struct-element-crash` (struct-inner crash in `container.array.append`, which routes to the scalar object path, not this array-inner branch). severity rationale: impact=silent-false-success × reach=every-session (generic `property.set` writer; object-array replace is a normal, not rare, authoring path) -> High.
