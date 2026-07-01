---
id: E-property-list-container-cpptype-type-erased
title: "property.list/property.get cppType emits bare 'TMap'/'TArray'/'TSet' with no key/value/element type — container parameter types are type-erased"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [property, container, map, discoverability]
---

# `property.list` / `property.get` `cppType` is type-erased for containers — the listing shows `TMap`, not `TMap<K,V>`

`property.list` and `property.get` report each UPROPERTY's `cppType` via the
shared helper `AddPropertyMetadataFields`
(`Handlers/Utility/UtilityPropertyHandler.cpp:533`), which calls
`Property->GetCPPType()` with default arguments. For container properties
(`FMapProperty`/`FArrayProperty`/`FSetProperty`) UE's `GetCPPType()` **only emits
the templated `TMap<K,V>` / `TArray<T>` / `TSet<E>` form when called with the
`ExtendedTypeText` out-param** (`GetCPPType(&ExtendedTypeText, ...)`); the no-arg
call this code makes returns the bare token `"TMap"` (likewise `"TArray"`,
`"TSet"`) with **no key/value/element type arguments**.

The same type-erasure independently affects a *second* emission site:
`system.inspect.inspect_class` serializes properties through
`PropertyToInspectJson` (`Utils/PropertyInspection.cpp:493`), which also calls the
no-arg `GetCPPType()`. (NOTE — an earlier draft of this ticket pinned the
`property.list` defect at `PropertyInspection.cpp:493`; that file:line is in fact
the `inspect_class` site, NOT `property.list`. `PropertyToInspectJson` is called
only from `EnvironmentHandler.cpp:1514`. The real `property.list`/`property.get`
site is `AddPropertyMetadataFields` at `UtilityPropertyHandler.cpp:533`. Both
sites share the same bug; the fix touches both.)

Consequence (ergonomic): a caller scanning `property.list` for a container
property can find that a property *is* a map/array/set, but **cannot tell what
key/value/element type it holds** from the listing — the listing shows the
unhelpful bare token `"TMap"`. The parameter type is only learnable by reading
engine/plugin source, on a tool whose entire job is to describe a property. The
type info is already on the `FProperty`; it is just being dropped by the no-arg
call.

This is a distinct surface from the two neighbouring tickets:
`F-container-map-value-type-coverage` is the **capability** gap (get/set can't
handle non-primitive values — that ticket, not this one, removes the
get/set-viability wall); `E-container-set-no-discoverable-target` is about
finding *where a container property lives*. Neither closes the gap that the
`cppType` `property.list` *does* emit is **type-erased**. This ticket is the
narrow ergonomic fix: make the descriptive output carry the container's parameter
types.

## Evidence (this task — `container.map.get_keys` focus, outcome=gap)

The friction note records the discovery cost verbatim: *"Struggled hard on
discovery. cppType from property.list is the bare token 'TMap' with NO key/value
type args, so you cannot tell a map's value type before calling get/set (and the
wiki's own recipe says scan cppType for 'TMap' but doesn't warn it's
type-erased). Probed ~18 Blueprint/asset CDOs finding zero TMaps before falling
back to asset.list full inventory + engine-source grep to locate UIKRetargeter's
top-level maps. Then container.map.get/set reject struct AND FText values with
UNSUPPORTED_VALUE_TYPE … I had to read the plugin source
(UtilityPropertyHandler.cpp, last resort) to confirm the supported set."*

Call-log shape (all the map discovery was pure overhead, 18 `property.list` +
4 `asset.list` calls, all `ok:true`):

- `property.list` ×18 across CDOs (`CE_Game`, `BP_DemoDisplay`, `PlayerCharacter`,
  `BP_DemoRoom`, several StateTree tasks, `BP_Echo_3rdPersonPawn`, `BP_Aera`,
  `UserInterfaceSettings`, `MyCharacter_UMG`, `BP_Gears`, `BP_EchoHair`,
  `BP_HairLgt`, the three `*InputBrushes` settings) — every map shown as bare
  `cppType: "TMap"`, so the value type was invisible at this stage.
- `asset.list` ×4 (DataAsset / Blueprint / PrimaryDataAsset / StateTree / full
  `/Game` inventory limit=2000) + an **engine-source grep** to find
  `UIKRetargeter`'s top-level maps.
- Only after `container.map.get`/`set` returned `UNSUPPORTED_VALUE_TYPE` (and a
  read of `UtilityPropertyHandler.cpp`) was the value type (`FIKRetargetPose`
  struct) and its unsupported-ness established.

If `property.list` had emitted `cppType: "TMap<FName, FIKRetargetPose>"` (or
structured `keyType`/`valueType` fields), the caller would have known at the
*first* listing that this map is struct-valued and that `get`/`set` cannot touch
it — collapsing ~18 probes + a source grep into a single read.

## What it should do

- Emit the **templated** cppType for containers: call
  `Property->GetCPPType(&ExtendedTypeText, 0)` and concatenate, so a map renders
  as `TMap<FName,FIKRetargetPose>`, an array as `TArray<FVector>`, a set as
  `TSet<FName>` — the type info is already available on the FProperty, it's just
  being dropped by the no-arg call. Non-container properties write nothing into
  the out-param, so their `cppType` is unchanged.
- Fix it once, in the shared path: `AddPropertyMetadataFields`
  (`UtilityPropertyHandler.cpp:533`) feeds both `property.list` (line 1625) and
  `property.get` (line 1459); fixing it there fixes both surfaces. Apply the same
  to the `system.inspect.inspect_class` site (`PropertyInspection.cpp:493`).
- (Optional, richer) structured `keyType` / `valueType` / `elementType` fields are
  a possible follow-up but out of scope for this ticket.

**Fix:** Add a shared exported helper `GetPropertyCppTypeWithParams(FProperty*)`
in `Utils/PropertyInspection.{h,cpp}` that passes the `ExtendedTypeText` out-param
to `GetCPPType` and concatenates, then route the three `cppType`-emitting sites
through it: `AddPropertyMetadataFields` (`UtilityPropertyHandler.cpp:533`, shared
by `property.list`/`property.get`) and `PropertyToInspectJson`
(`PropertyInspection.cpp:493`, used by `system.inspect.inspect_class`). Small,
localized; non-container `cppType` strings are unaffected (out-param stays empty).

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of a `container.map.get_keys`
  workflow task (outcome=gap; the value-type capability gap is separately filed as
  `F-container-map-value-type-coverage`, this ticket is the distinct DISCOVERY
  angle). `property.list` emits `cppType` via the no-arg `GetCPPType()`
  (`PropertyInspection.cpp:493`), which for `FMapProperty`/`FArrayProperty`/
  `FSetProperty` yields the bare token `"TMap"`/`"TArray"`/`"TSet"` with no
  key/value/element type — so a caller cannot tell a map's value type from the
  listing and only learns it by calling get/set (→`UNSUPPORTED_VALUE_TYPE`) or by
  reading source. This task spent 18 `property.list` probes + 4 `asset.list` +
  an engine-source grep on map discovery because every map showed as bare `"TMap"`.
  Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — `F-container-map-value-type-coverage`
  (capability), `E-container-set-no-discoverable-target` (target location;
  explicitly notes no container-kind/`cppType` filter), `F-property-list-name-filter`
  (name filter, DONE), `B-property-list-hides-reflected-props` (visibility of
  reflected props) — none cover the **type-erasure of the emitted container
  `cppType`**. Proposed: pass `ExtendedTypeText` to `GetCPPType` (and/or structured
  key/value/element type fields); interim docs caveat in `wiki-src/container.map.md`.
- `#2-reword` `OPEN` developer — Reworded: the symptom and fix concept are valid, but
  the root-cause file:line was wrong. `PropertyInspection.cpp:493`
  (`PropertyToInspectJson`) is consumed ONLY by `system.inspect.inspect_class`
  (`EnvironmentHandler.cpp:1514`), NOT by `property.list`. The real `property.list`
  (and `property.get`) `cppType` site is the shared helper `AddPropertyMetadataFields`
  at `UtilityPropertyHandler.cpp:533`. Retitled/rescoped to the narrow ergonomic
  "emit templated container cppType" fix; dropped the get/set-viability framing
  (dissolved by `F-container-map-value-type-coverage`) and the interim docs half
  (covered by `E-container-set-no-discoverable-target`).
- `#3-reword-fix` `IN-REVIEW` developer — Added shared exported helper
  `GetPropertyCppTypeWithParams(const FProperty*)` in
  `Utils/PropertyInspection.{h,cpp}` (passes `ExtendedTypeText` to
  `GetCPPType(&Ext, 0)` and concatenates → `TMap<FName,FString>` etc.; non-container
  types unaffected since the out-param stays empty). Routed all three `cppType`
  emitters through it: `AddPropertyMetadataFields`
  (`Handlers/Utility/UtilityPropertyHandler.cpp:533`, shared by `property.list` +
  `property.get`) and `PropertyToInspectJson` (`Utils/PropertyInspection.cpp:493`,
  `system.inspect.inspect_class`). Regression test
  `Tests/Utility/TestPropertyListContainerCppType.{h,cpp}`
  (`EditorAutomationRpcGateway.Property.List.ContainerCppType`): a fixture host with
  `TMap<FName,FString>`/`TArray<FVector>`/`TSet<FName>`/`float` UPROPERTYs drives the
  real `property.list` handler and asserts each container `cppType` carries the
  templated `<...>` parameters (fails on the pre-fix bare `"TMap"`/`"TArray"`/`"TSet"`),
  the scalar stays `float`, and `GetPropertyCppTypeWithParams` emits the templated
  map type directly.
