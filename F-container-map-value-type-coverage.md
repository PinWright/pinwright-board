---
id: F-container-map-value-type-coverage
title: "container.map.get/set only handle FStr/FInt/FFloat/FBool values — struct/object/FText/enum/FName-valued maps cannot be read or written by any RPC"
status: IN-REVIEW
severity: High
category: feature
tags: [container, map, value-type, struct, property]
---

# container.map.get/set cannot read or write any non-primitive map value

`container.map.get` and `container.map.set`
(`Handlers/Utility/UtilityPropertyHandler.cpp`, the `container.map.get` and
`container.map.set` handlers) only handle four reflected **value** types via a
`CastField` chain: `FStrProperty` / `FIntProperty` / `FFloatProperty` /
`FBoolProperty`. Every other value type — `FStructProperty`, `FObjectProperty`
(and soft/class variants), `FTextProperty`, `FNameProperty`, `FEnumProperty` /
`FByteProperty` enum — falls through to:

```
[UNSUPPORTED_VALUE_TYPE] Unsupported map value type.
```

There is **no sibling RPC** that closes this. The only map-value accessors are
`container.map.get`/`set`. `property.get`/`property.set` cannot index into a map
by key at all: their nested-path resolver `ResolveNestedPropertyPath`
(`Utils/PropertyInspection.cpp`) walks `FObjectProperty`/`FStructProperty` hops
and `FArrayProperty` `[N]` subscripts, but contains **no `FMapProperty`
traversal** (grep for `FMapProperty`/`ScriptMap` in that file returns nothing),
so there is no `Map.Key` / `Map["Key"]` form. The result: a designer auditing or
tuning a reflected `TMap` UPROPERTY can enumerate its keys
(`container.map.get_keys`, `container.map.has_key` both work) but cannot read or
edit any entry whose value is a struct, object, text, name, or enum — which is
the common case for real config/gameplay maps. In this Content Examples project
**every** reflected host map encountered is struct-, FText-, object-, or
name-valued; none expose a primitive value the handler accepts, so the
read/edit-a-map-entry workflow is impossible end-to-end on real assets here.

This is the read-side analogue of the map-value-type gap and parallels the
already-tracked container shortfalls on the key/element side
(`B-container-set-fname-lookup-broken` — set lookup loop missing the FName
branch) but is a distinct surface: those are membership-match defects on
existing handlers; this is the **absence** of any way to (de)serialize a
non-primitive map *value*. `container.array.set` shares the same primitive-only
inner-type restriction (`UNSUPPORTED_TYPE` for non-FStr/FInt/FFloat/FBool inner),
so a fix should ideally generalize across both containers.

## What it should do

`container.map.get`/`set` (and ideally `container.array.get`/`set`) should
(de)serialize the full reflected value-type vocabulary, reusing the property's
own import/export machinery rather than the hand-rolled four-type `CastField`
chain:

- **Read** — export the value via `ExportPropertyToJsonValue(ValuePtr, ValueProp)`
  (the same exporter `property.get`/`property.list` already use for structs,
  objects, text, enums), so a struct value comes back as a structured JSON
  object instead of erroring.
- **Write** — apply the JSON value via `ApplyJsonValueToProperty(TempValue,
  ValueProp, ValueField, Error)` (the converter `property.set` already uses),
  so a JSON object/string/number coerces into the struct/object/text/enum value
  per the property's type, instead of erroring on anything but the four
  primitives.

That collapses the per-type branch into the shared, already-tested
export/import path and lets the map verbs cover whatever `property.get`/`set`
cover. For value types that genuinely cannot round-trip, keep a precise
`UNSUPPORTED_VALUE_TYPE` (naming the cppType) rather than rejecting all
non-primitives.

**Workaround (today):** none for editing a struct-valued map entry in place. A
caller can only read/overwrite the *entire* map UPROPERTY via
`property.get`/`property.set` on the whole `TMap` (whole-container set, the very
pattern the per-key verbs exist to avoid), and even that depends on the value
type round-tripping through whole-property set.

## Repro (verbatim, replay-confirmed live)

Real host asset, FName-keyed map whose value type is the struct
`FIKRetargetPose`:

- `container.map.get_keys { objectPath:
  "/Game/Characters/Mannequin_UE4/Rigs/RTG_UE4Manny_UE5Manny.RTG_UE4Manny_UE5Manny",
  propertyName: "TargetRetargetPoses" }`
  → `{ keys: ["Default Pose","Manny Retarget Pose","Quinn Retarget Pose"],
  keyCount: 3 }` (works)
- `container.map.has_key { …, key: "Default Pose" }` → `hasKey: true` (works)
- `container.map.get { …, key: "Default Pose" }`
  → `[UNSUPPORTED_VALUE_TYPE] Unsupported map value type.`
- `container.map.set { …, key: "Audit Probe Pose", value: "probe" }`
  → `[UNSUPPORTED_VALUE_TYPE] Unsupported map value type.`

`get_keys`/`has_key` confirm the map and the entry are real and resolvable; only
the value (de)serialization is the wall.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed on the live editor: on the
  real asset `/Game/Characters/Mannequin_UE4/Rigs/RTG_UE4Manny_UE5Manny`,
  `TargetRetargetPoses` (FName→`FIKRetargetPose`), `container.map.get_keys` and
  `container.map.has_key` succeed but `container.map.get` and `container.map.set`
  both return `[UNSUPPORTED_VALUE_TYPE] Unsupported map value type.`. Root cause
  read in source: the get/set handlers' value branch handles only
  `FStr`/`FInt`/`FFloat`/`FBool` (`UtilityPropertyHandler.cpp`); no `FMapProperty`
  traversal exists in `ResolveNestedPropertyPath` (`PropertyInspection.cpp`), so
  `property.get`/`set` offer no map-key alternative. No sibling RPC reads/writes a
  non-primitive map value. Dedup: ripgrep across OPEN/DONE/WONTFIX — no ticket
  covers map value-type coverage; `B-container-set-fname-lookup-broken` is a set
  membership-match defect (FName *element* matching), `E-container-set-no-discoverable-target`
  is set-target discoverability, both distinct. Proposed fix: route value read
  through `ExportPropertyToJsonValue` and value write through
  `ApplyJsonValueToProperty`, generalizing across `container.array.get`/`set` too.
- `#2-corroborating-audit` `OPEN` reporter — Cross-task evidence (process audit of a
  second `container.map.get_keys` workflow task, independent run). Same wall hit on
  the same asset reached via a fresh discovery path: `RTG_UE4Manny_UE5Manny`
  `TargetRetargetPoses` (FName→`FIKRetargetPose`) — `get_keys`/`has_key` succeed,
  `container.map.get` and `container.map.set` both return `[UNSUPPORTED_VALUE_TYPE]
  Unsupported map value type.`, so step-6 insert and the set/get round-trip are
  impossible. Confirms the report that **this Content Examples project exposes no
  primitive-valued TMap at all** — every host map is struct-/FText-/object-/name-valued
  — so the read/edit-a-map-entry workflow cannot complete end-to-end on any real asset
  here, reinforcing the High severity. The DISCOVERY cost that fed this (18 `property.list`
  CDO probes + engine grep, because `cppType` is the type-erased bare `"TMap"`) is filed
  separately as the ergonomic ticket `E-property-list-container-cpptype-type-erased`.
- `#3-fix-value-type-coverage` `IN-REVIEW` developer — Replaced the hand-rolled
  four-type (FStr/FInt/FFloat/FBool) CastField chains in all four container value/element
  handlers with the shared export/import path property.get/property.set already use, so
  the container verbs now cover whatever the property verbs cover (struct, object,
  soft-object, FText, FName, enum). File:
  `Handlers/Utility/UtilityPropertyHandler.cpp`.
  * `container.map.get` value branch → `ExportPropertyToJsonValue(PairPtr, ValueProp)`.
    The exporter takes a CONTAINER base and re-derives the value via
    `ValueProp->ContainerPtrToValuePtr`; UE sets `ValueProp->Offset_Internal ==
    MapLayout.ValueOffset` (PropertyMap.cpp) and `GetValuePtr(i) == GetPairPtr(i) +
    ValueOffset`, so the PAIR pointer is the correct container (passing GetValuePtr
    directly — as the ticket's literal `ExportPropertyToJsonValue(ValuePtr, …)` would —
    double-adds the offset; all three validity lenses flagged this). Struct values now
    come back as structured JSON objects.
  * `container.map.set` value branch → `FindOrAdd(TempKey)` (returns the value ptr;
    pair base = value − `ValueProp->GetOffset_ForInternal()`) then
    `ApplyJsonValueToProperty(PairPtr, ValueProp, …)`. A freshly inserted key whose value
    write fails is rolled back (RemoveAt + Rehash) so no phantom default entry is left.
    The dead TempValue buffer was removed (FindOrAdd default-constructs the value
    in-place). Minor consistency change: a numeric JSON value into an FString-valued map
    now errors (the shared converter rejects number→string) instead of coercing via %g —
    this matches whole-property `property.set` behavior.
  * `container.array.get`/`set` element branches → same helpers on the element pointer
    directly (FArrayProperty leaves `Inner->Offset_Internal == 0`, so the element ptr IS
    a valid container), generalizing the fix across both containers as the ticket asked.
  Regression test: `Tests/Utility/TestContainerValueTypeCoverage.cpp` +
  `TestContainerValueTypeHost.h` (a transient UObject hosting `TMap<FName,
  FTestContainerPose>` — mirroring the repro's `TMap<FName, FIKRetargetPose>` — plus a
  `TMap<FString, enum>`, a `TArray<struct>`, and a `TArray<FName>`). It drives the real
  registered handlers end-to-end: map.set/get of a struct value (asserting structured
  JSON back), an enum value, and array.set/get of a struct element and an FName element.
  Reverting to the four-type chain makes every set return UNSUPPORTED_VALUE_TYPE /
  UNSUPPORTED_TYPE and the test fails. Not yet compiled/run — left for the test phase.
