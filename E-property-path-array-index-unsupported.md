---
id: E-property-path-array-index-unsupported
title: "property.get/set nested paths can't index an array element (`Keys[2].KeyType.BaseClass`, `ResultsStructs[0].Asset`) — `[N]` segment returns PROPERTY_NOT_FOUND, forcing a fallback to the named auto-subobject"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [property, path-resolver, array-index, nested-path, readback, docs]
---

# `property.*` nested paths support struct/object hops but not array-element subscripts

`property.get`/`property.set` advertise dotted nested `propertyName` paths and
their resolver (`ResolveNestedPropertyPath`, `Utils/PropertyInspection.cpp:26`)
hops `FStructProperty` (into a struct member) and `FObjectProperty` (into a
subobject) segments — but it has **no array-element subscript segment**. The
moment a path contains `Keys[2]`, `ResultsStructs[0]`, etc., resolution fails at
that segment with `[PROPERTY_NOT_FOUND] … Property 'Keys[2]' not found in scope
'BlackboardData' (segment 1 of 3)`. There is no way to address one element of a
`TArray<…>` (or its sub-fields) through the generic `property.*` read route, even
though indexing into an array is the most natural way to verify a single
authored element.

This is the documented "natural next call" an agent reaches for after a
whole-array read returns opaque/empty elements (the array-read surface hands back
`[{},{},{}]` or omits per-element detail), and it dead-ends. The agent is then
forced to discover and address the engine's **named auto-generated subobjects**
instead (e.g. `…:BlackboardKeyType_Object_1`, the per-key `BlackboardKeyType_*`
inner objects) — a non-obvious naming scheme that requires `asset.dump` or trial
reads to learn, when `Keys[1].KeyType.BaseClass` would have been the obvious path.

## Cross-task evidence (this is an aggregation of a recurring, carved-out gap)

- **Guard-AI blackboard task** (`ai.create_blackboard_asset`, this audit; outcome
  was a tool bug filed separately as `B-add-blackboard-key-base-object-class-dropped`).
  To confirm whether `baseObjectClass` had been dropped on the `TargetActor`
  Object key, the agent tried the obvious array-index paths and both failed:
  - `property.get { objectPath:"/Game/AI/Blackboards/BB_GuardAI.BB_GuardAI", propertyName:"Keys[2].KeyType.BaseClass" }`
    → `[PROPERTY_NOT_FOUND] … Property 'Keys[2]' not found in scope 'BlackboardData' (segment 1 of 3)`
  - `property.get { … propertyName:"Keys[0].KeyType.BaseClass" }`
    → same `PROPERTY_NOT_FOUND` on `Keys[0]`.

  Only after those two dead calls did the agent fall back to reading the named
  subobjects directly — `property.get { objectPath:"…BB_GuardAI:BlackboardKeyType_Object_1", propertyName:"BaseClass" }`
  — which worked and exposed the dropped class. Friction note (verbatim):
  *"array-index property paths (Keys[2].KeyType.BaseClass) error with
  PROPERTY_NOT_FOUND … I only nailed it down by reading the named
  BlackboardKeyType_Object_N subobjects directly via property.get."*

- **Chooser table** (`B-instanced-struct-export-opaque`, IN-REVIEW). Same resolver,
  same shape: `property.get { … propertyName:"ResultsStructs[0].Asset" }` →
  `[PROPERTY_NOT_FOUND] Failed to resolve nested property path 'ResultsStructs[0].Asset':
  Property 'ResultsStructs[0]' not found in scope 'ChooserTable' (segment 1 of 2)`.
  That ticket's `#2` developer note **explicitly carves this out**: *"the `#1`
  nested-path footnote `ResultsStructs[0].Asset → [PROPERTY_NOT_FOUND]` is a
  separate `UtilityPropertyHandler` path-resolver gap and is intentionally out of
  scope here."* — i.e. acknowledged but never given its own ticket. This ticket is
  that ticket.

The two reports are on different asset classes (`UBlackboardData`,
`UChooserTable`) and different scope errors but one root cause: the nested-path
resolver does not parse/handle an `[N]` array subscript segment.

## What it should do / how to fix

**Docs-first (cheap win, ships now, no build):** amend `docs/wiki-src/property.md`
so the `propertyName` nested-path description states plainly what the resolver
*does* and *does not* support: dotted struct-member and subobject hops are
supported, **array-element indexing (`Foo[N]`) is NOT** — a path containing `[N]`
fails `PROPERTY_NOT_FOUND` at that segment. Point agents at the working
read-backs: read the whole array property and index client-side, or address the
engine's named auto-subobject directly (for blackboard Object keys, the per-key
`…:BlackboardKeyType_Object_<N>` inner object carries `BaseClass`; discover the
exact name via `asset.dump`). The page today says only "supports nested paths
with dots … For array properties, pass a JSON array; the call replaces the entire
array" and never tells the reader that an `[N]` segment is rejected, so the
guess is reasonable and the failure is surprising.

**Optional resolver enhancement (larger, separate):** teach
`ResolveNestedPropertyPath` (`Utils/PropertyInspection.cpp`) to parse a trailing
`[N]` on a segment and, when the resolved property is an `FArrayProperty`, use a
`FScriptArrayHelper` to bounds-check and step into element `N` (then continue
resolving the remaining `.field` segments against the element's inner property).
This would make `Keys[2].KeyType.BaseClass` and `ResultsStructs[0].Asset` resolve
directly and benefits every `property.*` consumer at the shared locus.

**Workaround (today):** read the full array via `property.get` (no `[N]`) and pick
the element yourself; or, for blackboard key base classes, `property.get` the
named `…:BlackboardKeyType_Object_<N>` subobject's `BaseClass` (name discoverable
via `asset.dump`).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `ai.create_blackboard_asset` guard-AI task (38 calls; outcome clean apart from the separately-filed write bug `B-add-blackboard-key-base-object-class-dropped`). PROCESS friction distinct from any write bug and from the perception/controller readback thinness in `E-get-ai-info-no-perception-readback`: the generic `property.*` nested-path resolver hops struct/object segments but rejects an array-element `[N]` subscript, so the obvious verification paths `Keys[2].KeyType.BaseClass` and `Keys[0].KeyType.BaseClass` both returned `[PROPERTY_NOT_FOUND]` (2 dead calls) and the agent had to fall back to reading the engine's named auto-subobject `…:BlackboardKeyType_Object_1` directly (a non-obvious naming scheme it had to discover via `asset.dump`). Cross-task aggregation: the same `[N]`-unsupported gap was hit on a `UChooserTable` (`property.get ResultsStructs[0].Asset → PROPERTY_NOT_FOUND`) in `B-instanced-struct-export-opaque`, whose `#2` note explicitly declares the array-index path-resolver failure out of scope for that ticket — leaving it unowned until now. Docs overlay to improve: `docs/wiki-src/property.md` (state nested paths support struct/subobject hops but NOT `Foo[N]` indexing; name the whole-array-read and named-subobject workarounds). Optional larger fix: add `FScriptArrayHelper`-based `[N]` element stepping to `ResolveNestedPropertyPath` (`Utils/PropertyInspection.cpp`). Workaround: read the whole array and index client-side, or `property.get` the named `…:BlackboardKeyType_Object_<N>` subobject's `BaseClass`.
- `#3-resolver-fix` `IN-REVIEW` developer — Implemented the root-cause resolver enhancement at the shared locus, not just the docs steer. `Utils/PropertyInspection.cpp::ResolveNestedPropertyPath` now parses an array-element subscript on a path segment in BOTH forms the evidence hit — the bracket form `Keys[2].KeyType.BaseClass` and the dotted-numeric form `SensesConfig.0.PeripheralVisionAngleDegrees` — and steps into element N via `FScriptArrayHelper` (bounds-checked) before continuing to resolve the remaining `.field` segments against the element's inner struct/object. The fix fits the existing `(FProperty* leaf, void*& container)` return contract WITHOUT touching any of the 10+ shared call sites: an `FArrayProperty`'s inner property has `Offset_Internal == 0`, so handing back `(ArrayProp->Inner, Helper.GetRawPtr(N))` lets every consumer's `ContainerPtrToValuePtr` / `ExportPropertyToJsonValue` / `ApplyJsonValueToProperty` address the element value directly — so the adversarial "signature can't express element N" concern does not apply here. Error paths are explicit: out-of-range index (`… out of range … (length N)`), subscripting a non-array (`… is not an array …`), and a bare array hop with no index (`… without an element index (use 'Keys[N]' or 'Keys.N.<field>')`). Files: `Source/EditorAutomationRpcGateway/Private/Utils/PropertyInspection.cpp` (added `ParsePathSegment` / `IsNumericIndexSegment` / `StepIntoArrayElement` helpers and the array-index branches in the traversal loop); docs overlay `Docs/wiki-src/property.md` (documents the now-supported `Foo[N]` / `Foo.N.field` subscript forms under `property.set`, names the affected verbs, and keeps the opaque-`[{}]` object-array note for `actor.get_component_property`'s `SensesConfig`). Regression test: appended `FResolveNestedPropertyPathArrayIndexTest` (`EditorAutomationRpcGateway.utils.property_inspection.ArrayIndexNestedPath`) to `Source/EditorAutomationRpcGateway/Private/Tests/Utility/TestPropertyUtilsArrayStructElement.cpp` — reuses the existing `UTestArrayStructElementHost` (`TArray<FStruct>` fixture) and drives the production `ResolveNestedPropertyPath` for `Items[1].IntField`, `Items.0.StringField`, the whole-struct-element `Items[0]`, plus the out-of-range / non-array / no-index error cases; if the `[N]`/`.N.` stepping is reverted these reads resolve to null and the test fails.
- `#2-cross-verb-get-component-property` `OPEN` reporter — Cross-task aggregation: the SAME `ResolveNestedPropertyPath` array-element gap was hit through a THIRD verb and a DIFFERENT index syntax, on a guard-AI perception-verify task (struggle audit, ~29 calls). After `actor.get_component_property {componentName:AIPerception, propertyName:SensesConfig}` returned an opaque `[{}]` (object-array elements serialize without per-element fields), the agent reached for the obvious single-leaf read `actor.get_component_property {... propertyName:"SensesConfig.0.PeripheralVisionAngleDegrees"}` and it dead-ended: `[NOT_FOUND] Property 'SensesConfig.0.PeripheralVisionAngleDegrees' not found on component 'AIPerception': Cannot traverse into property 'SensesConfig' of type 'ArrayProperty'`. Significance: `actor.get_component_property` now routes dotted `propertyName` through the same shared `ResolveNestedPropertyPath` (`Utils/PropertyInspection.cpp`) that `property.get`/`property.set` use (per the IN-REVIEW fix in `E-get-component-property-no-subfield-select` `#2`), so this is the identical root-cause gap surfacing on a new verb — and via the dotted-numeric `Foo.N.field` form, NOT just the `Foo[N]` bracket form the `#1` evidence used. The error here is raised at `PropertyInspection.cpp:104` (`Cannot traverse into property '%s' of type '%s'`), the resolver's "non-struct/non-object segment" branch — confirming neither `[N]` brackets NOR `.N.` dotted-numeric indices are parsed as array subscripts. The proposed `FScriptArrayHelper`-based `[N]` element-stepping fix should therefore ALSO accept (or normalize) a `.N.` numeric segment, and benefits this third verb at the shared locus. The agent could not read the live `PeripheralVisionAngleDegrees=90` leaf at all (it equals the engine default so it was elided from the `blueprint.scs.get` SCS dump too) and had to confirm it via the `configure_sight_config` return value + the handler source — so the array-element read gap directly blocked a single-field readback that no other route covered. Friction note verbatim: *"PeripheralVisionAngleDegrees=90 was elided from the SCS dump (equals engine default) and could not be read as a live leaf (array-element traversal unsupported)."* Reinforces that the docs steer on `docs/wiki-src/property.md` should also cover the dotted-numeric form and name `actor.get_component_property` (now sharing the resolver) as an affected verb, and that the overlay should warn that an object-array (`TArray<U...*>`, e.g. `SensesConfig`) read returns opaque `[{}]` elements with no per-element drill-in.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
