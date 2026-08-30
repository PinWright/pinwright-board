---
id: B-asset-dump-property-unsupported-sentinel-on-common-types
title: "asset.dump properties.json emits `{_kind:unsupported, cpp_type:<T>}` sentinels for common FProperty types (uint32/int8/uint16/uint64, TWeakObjectPtr, TLazyObjectPtr, TOptional)"
status: DONE
severity: High
category: bug
tags: [asset-dump, properties, property-utils, fproperty, transient, serializer]
---

# `asset.dump` properties.json emits opaque `unsupported` sentinels for common FProperty kinds

`ExportPropertyToJsonValue()` in
`Source/PinWright/Private/Utils/PropertyExport.cpp` (`:605`)
covers `FFloatProperty`, `FDoubleProperty`, `FIntProperty`,
`FInt64Property`, `FByteProperty`, and `FEnumProperty` on the numeric
path, plus `FObjectProperty`/`FClassProperty`/`FInterfaceProperty` on
the object-ref path. Everything else falls through the chain and hits
the terminal fallback at the bottom of the function:

```cpp
// PropertyExport.cpp:1118-1126 (as filed: PropertyUtils.cpp ~812-819)
TSharedPtr<FJsonObject> Obj = MakeShared<FJsonObject>();
Obj->SetStringField(TEXT("_kind"), TEXT("unsupported"));
Obj->SetStringField(TEXT("cpp_type"), Property->GetCPPType(nullptr, CPPF_None));
return MakeShared<FJsonValueObject>(Obj);
```

*(One line of the block has since changed: `cpp_type` is now built by
`GetPropertyCppTypeWithParams(Property)` at `PropertyExport.cpp:1123` — helper at
`Utils/PropertyInspection.cpp:8`, declared `Utils/PropertyInspection.h:81` — which
is the fix for the bare-`TOptional` no-inner-type case `#1` reported. The fenced
snippet is left as filed.)*

Across the current full `App/` asset-dump sweep this fires **10,807
times**, dominated by very common UE FProperty subclasses:

| cpp_type | Count | What it is |
|---|---:|---|
| `uint32` | 9,638 | `FUInt32Property` — transient runtime counters on UPrimitiveComponent/USkeletalMeshComponent (e.g. `CacheMeshDescriptionTrianglesCount`, `CacheMeshDescriptionVerticesCount`) |
| `int8` | 573 | `FInt8Property` — `VirtualTextureCullMips`, `VirtualTextureLodBias`, `VirtualTextureMinCoverage` on UPrimitiveComponent |
| `TWeakObjectPtr<APhysicsVolume>` | 428 | `FWeakObjectProperty` — `USceneComponent::PhysicsVolume` runtime cache (transient) |
| `TOptional<...>` | 77 | `FOptionalProperty` (UE 5.5+) — includes a *bare* `cpp_type: "TOptional"` with no inner template arg, which is a separate template-arg parser regression |
| `TLazyObjectPtr<ALODActor>` | 47 | `FLazyObjectProperty` — HLOD owner ref |
| `uint16` | 36 | `FUInt16Property` — `USkeletalMeshComponent::CachedAnimCurveUidVersion` |
| `TWeakObjectPtr<USkinnedMeshComponent>` | 5 | `FWeakObjectProperty` — `USkinnedMeshComponent::LeaderPoseComponent` |
| `uint64` | 3 | `FUInt64Property` |

Two distinct gaps in the serializer:

1. **Missing numeric encoders.** `FUInt32Property`, `FInt8Property`,
   `FUInt16Property`, `FUInt64Property` are reflected, have integer
   storage, and trivially encode as JSON numbers via
   `FNumericProperty::GetSignedIntPropertyValue()` or the unsigned
   equivalents. Nothing prevents handling them next to
   `FIntProperty`/`FInt64Property`, now `PropertyExport.cpp:649-658`
   (as filed: lines 442–451).

2. **Missing weak/lazy/optional object dispatch.** `FWeakObjectProperty`
   and `FLazyObjectProperty` are direct siblings of `FObjectProperty`
   (all derive from `FObjectPropertyBase`) and resolve to a `UObject*`
   via `GetObjectPropertyValue_InContainer()`. They should follow the
   same path-string-or-null shape as `FObjectProperty`, now
   `PropertyExport.cpp:737-759` (as filed: lines 500–510). Specifically,
   `ExportObjectRefElement` (now `PropertyExport.cpp:347-408`; as filed:
   PropertyUtils.cpp ~124-173) handles `FObjectProperty`, `FClassProperty`,
   `FInterfaceProperty`, and `FSoftObjectProperty` but lacks an
   `FWeakObjectProperty` branch — add one mirroring the `FObjectProperty`
   path, dereferencing via `WeakProp->GetObjectPropertyValue` /
   `FWeakObjectPtr::Get` (returns null on stale handles). `FOptionalProperty`
   (UE 5.5+) wraps an inner property and needs a small wrapper handler.

## Widget tree.xml manifestation

The same fallthrough surfaces in widget `tree.xml` dumps. When a UMG
widget exposes a `TWeakObjectPtr<U...>` UPROPERTY that differs from the
CDO, the tree.xml emitter writes the sentinel JSON object flattened to
attribute form:

```
LinkedSwitcher="{_kind=unsupported,cpp_type=TWeakObjectPtr<UCommonAnimatedSwitcher>}"
HandlerComponent="{_kind=unsupported,cpp_type=TWeakObjectPtr<USKGMLEHandlerComponent>}"
```

Observed in CommonUI tab panels (`W_ControllerAxesPanel`,
`W_ControllerAxisSelectPanel`, `W_ControllerRangeCalibratePanel` —
`LinkedSwitcher`) and MapEditor widgets (`W_AppMapEditor_ActionPanel`,
`W_AppMapEditor_ActionPanelTop`, `W_TransformWidget`,
`W_TransformWidgetLine` — `HandlerComponent`). Note: a fresh
`asset.dump_folder` may not reproduce all occurrences — the offending
attributes are skipped entirely when the runtime value matches the CDO,
which is the common case for weak runtime back-references populated at
`Construct` time. The defect is still latent.

Emission path: `WidgetXmlExporter::CollectOverriddenAttributes`
(`WidgetXmlExporter.cpp:236`) calls `ExportPropertyToJsonValue`, then
`WidgetXmlHelpers::JsonValueToAttrString` (`WidgetXmlUtils.h:~148-159`)
flattens the JSON object into the visible `{_kind=unsupported,...}`
attribute literal. The fix above (add `FWeakObjectProperty` branch to
`ExportObjectRefElement`) resolves this manifestation automatically —
weak refs become path strings identical to strong object-ref attributes
(e.g. `Background.ResourceObject="/Script/.../MaterialInstanceConstant'/Game/...'"`),
without cycle risk (weak refs are cycle-safe by construction).

## Why it matters

- 10.8k opaque entries per sweep is the single largest noise source in
  the asset dumps; it makes JSON diffs and consumer parsers churn on
  values that are trivially representable.
- Consumers (LLM agents, asset-dump readers) cannot distinguish a real
  cached value from `unknown` — they have to fall back to `nullptr`
  semantics, which is wrong for numeric counters.
- The bare-`TOptional` cpp_type at
  `App/App/Drone/PioneerGoPro/drone_GoPro/properties.json` line ~370
  (`CachedHasVertexColors` inside `SourceModels[]`) suggests the
  template-arg parser in `GetCPPType()`/the call site loses the inner
  type spec on `FOptionalProperty` — worth confirming.

## Repro

1. `call("asset.dump", {"assetPath": "/App/App/B_RaceAnalyzerPath"})`
2. Open the resulting
   `.editor-automation/asset-dumps/App/App/B_RaceAnalyzerPath/properties.json`
3. Observe `PhysicsVolume` →
   `{"_kind":"unsupported","cpp_type":"TWeakObjectPtr<APhysicsVolume>"}`
   (lines ~275, ~552) and `VirtualTextureCullMips`/`LodBias`/`MinCoverage`
   →
   `{"_kind":"unsupported","cpp_type":"int8"}` (lines ~311–323).
4. For the bare-TOptional case: `call("asset.dump", {"assetPath": "/App/App/Drone/PioneerGoPro/drone_GoPro"})`
   → `properties.json:370` shows
   `{"_kind":"unsupported","cpp_type":"TOptional"}` on `SourceModels[].CachedHasVertexColors`.

## Impact

- High volume (~10.8k occurrences / sweep) — biggest single contributor
  to opaque entries in properties.json.
- Cross-cutting: hits every USkeletalMesh, USkeletalMeshComponent,
  USceneComponent, UPrimitiveComponent — most actor BPs are affected.
- Most affected props are `UPROPERTY(Transient)` runtime caches; the
  cleanest fix is upstream (skip transient), so the symptom shrinks
  drastically even if the per-type encoders are deferred.

**Fix:** Two complementary fixes in
`Source/PinWright/Private/Utils/PropertyExport.cpp`:

- **Skip `CPF_Transient` UProperties** in the top-level dump walk (not
  in `ExportPropertyToJsonValue` itself — at the call site that drives
  per-property emission, `PropertyExport.cpp:1460-1465`; as filed: near
  `ExportPropertyToJsonValue` line 1961).
  This single change eliminates the `PhysicsVolume` weak refs, the
  `CacheMeshDescription*` counters, the
  `CachedAnimCurveUidVersion`, and the bulk of the int8 virtual-texture
  fields in one stroke, mirroring the precedent in
  `B-asset-dump-transient-controller-ref-leaks` (#3 DONE) where
  transient object refs are already suppressed for /Engine/Transient
  targets. The `flags` array in properties.json already exposes
  `"Transient"` so consumers can verify which fields were skipped.
- **Add typed encoders for the remaining numeric / smart-ptr property
  classes** so that non-transient occurrences round-trip correctly:
  - `FInt8Property`, `FInt16Property`, `FUInt16Property`,
    `FUInt32Property`, `FUInt64Property` → `JsonValueNumber` via
    `FNumericProperty::GetSignedIntPropertyValue()` /
    `GetUnsignedIntPropertyValue()` (cast `uint64` to double when it
    fits; emit as decimal string with `_kind: "uint64"` past
    2^53 to avoid double precision loss).
  - `FWeakObjectProperty`, `FLazyObjectProperty` →
    same path-string-or-null shape as `FObjectProperty`, now
    `PropertyExport.cpp:737-759` (as filed: lines 500–510),
    via `GetObjectPropertyValue_InContainer()`.
  - `FOptionalProperty` (UE 5.5+, guarded by `__has_include` or version
    macro) → recurse on `GetValueProperty()` when `IsSet()`, else
    `null`; on the bare-TOptional regression, also confirm that the
    sentinel `cpp_type` carries the inner template arg (probably
    requires passing the property type-text params, not raw `CPPF_None`,
    to `GetCPPType()`).

## History
- `#1-initial-repro` `OPEN` reporter — Full `App/` asset-dump sweep contains 10,807 `{_kind:"unsupported", cpp_type:<T>}` sentinels emitted by the terminal fallback at `PropertyUtils.cpp:812-819`. Breakdown: `uint32` 9,638 / `int8` 573 / `TWeakObjectPtr<APhysicsVolume>` 428 / `TOptional` 77 (incl. one bare `TOptional` no-inner-type at `App/App/Drone/PioneerGoPro/drone_GoPro/properties.json` line ~370) / `TLazyObjectPtr<ALODActor>` 47 / `uint16` 36 / `TWeakObjectPtr<USkinnedMeshComponent>` 5 / `uint64` 3. Two root causes: (a) `ExportPropertyToJsonValue` (PropertyUtils.cpp:398) lacks dispatch for `FUInt32Property`, `FInt8Property`, `FUInt16Property`, `FUInt64Property`, `FWeakObjectProperty`, `FLazyObjectProperty`, `FOptionalProperty`; (b) most hits are `UPROPERTY(Transient)` runtime caches on UPrimitiveComponent/USkeletalMeshComponent/USceneComponent that shouldn't be dumped at all. Preferred fix is skip-Transient at the walk site + add the missing typed encoders for non-transient occurrences. Live-reproducible (not stale cache).
- `#2-merged-tree-xml-weak-ref-ticket` `OPEN` reporter — Absorbed a parallel ticket (filed minutes earlier in the same triage session) covering the widget `tree.xml` manifestation: `{_kind=unsupported,cpp_type=TWeakObjectPtr<...>}` flattened as XML attribute literal in 7 CommonUI/MapEditor widgets (`LinkedSwitcher` on `W_ControllerAxesPanel`/`W_ControllerAxisSelectPanel`/`W_ControllerRangeCalibratePanel`; `HandlerComponent` on `W_AppMapEditor_ActionPanel`/`W_AppMapEditor_ActionPanelTop`/`W_TransformWidget`/`W_TransformWidgetLine`). Same `FWeakObjectProperty` dispatch gap, surfacing through `WidgetXmlExporter::CollectOverriddenAttributes` → `ExportPropertyToJsonValue` → `JsonValueToAttrString`. Adding the `FWeakObjectProperty` branch to `ExportObjectRefElement` (PropertyUtils.cpp ~124-173) — already required by gap (2) — resolves both surfaces. Source duplicate ticket file deleted.
- `#3-typed-encoders-and-transient-skip` `IN-REVIEW` developer — Added numeric encoders (FInt8/FInt16/FUInt16/FUInt32/FUInt64), Weak/Lazy/Optional object dispatch in PropertyUtils.cpp; added CPF_Transient skip at BuildClassPropertyJson walk site; new regression tests in TestAssetDumpNumericAndSmartPointerProperties.cpp covering all new branches and the transient-skip behavior.
- `#4-verify-source-and-tests` `DONE` tester — Verified by source + test inspection (editor offline so couldn't run live asset.dump). PropertyUtils.cpp has FInt8/FInt16/FUInt16/FUInt32/FUInt64 numeric encoders at lines 474-500, FWeakObjectProperty/FLazyObjectProperty dispatch at 604-619, FOptionalProperty handler at 626+ (guarded by EDITORAUTOMATIONRPCGATEWAY_HAS_OPTIONAL_PROPERTY), and CPF_Transient | CPF_DuplicateTransient skip at the BuildClassPropertyJson walk site (lines 2104-2111). All 9 tests in TestAssetDumpNumericAndSmartPointerProperties.cpp directly exercise the new branches (UInt32/Int8/UInt16/UInt64 → EJson::Number with value preserved; WeakRef populated → path string; WeakRef/LazyRef null → EJson::Null; Optional set → inner number; Optional unset → null; TransientPropertyWalkSkip → TransientCounter absent, PersistentCounter present). Header counterfactual documents which assertions break on revert.
- `#5-live-asset-dump-verify` `DONE` tester — Ran live `asset.dump` on both Repro assets. `/App/App/B_RaceAnalyzerPath/properties.json`: 0 occurrences of `_kind:unsupported`; `PhysicsVolume` (TWeakObjectPtr) now serializes as `null` at lines 275/536/842/1131; `VirtualTextureCullMips`/`VirtualTextureLodBias`/`VirtualTextureMinCoverage` (int8) now serialize as integer `0` at lines 307-309/585-587/874-876/1163-1165 — previously each emitted `{_kind:"unsupported",cpp_type:"int8"}` or `cpp_type:"TWeakObjectPtr<APhysicsVolume>"`. `/App/App/Drone/PioneerGoPro/drone_GoPro/properties.json`: 0 sentinels; bare-TOptional regression on `CachedHasVertexColors` (line 195) now serializes as `null` instead of `{_kind:"unsupported",cpp_type:"TOptional"}`. Frontmatter flipped IN-REVIEW → DONE; `#4` had marked DONE prematurely on source-only inspection (protocol forbids that), this bullet ratifies with live observation.
- `#6-repoint-citations-after-property-utils-split` `DONE` reporter — Citation maintenance only; **no behavioural claim changes and the status is untouched**. `Utils/PropertyUtils.cpp` was split into `PropertyExport.cpp` / `PropertyImport.cpp` / `PropertyInspection.cpp` / `PropertyDiff.cpp` (`PropertyUtils.h` survives only as a deprecated umbrella forwarder), and the module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/`, so this ticket's eleven `PropertyUtils.cpp` citations — the largest cluster on the board — were unresolvable paths a fixer could not open, not stale line numbers. **Every one of them lands in `PropertyExport.cpp`**; none in `PropertyImport`, `PropertyInspection` or `PropertyDiff`, and nothing was lost in the split. Each repointed line was read and verified at plugin HEAD `ef8a1f1b`. Body citations are repointed in place with the as-filed numbers kept beside them. History rows `#1`-`#5` are left verbatim per the append-only rule, so their citations map as follows. `#1`: terminal `unsupported` fallback `PropertyUtils.cpp:812-819` → `PropertyExport.cpp:1118-1126`; `ExportPropertyToJsonValue` `:398` → `:605`. `#2`: `ExportObjectRefElement` `~124-173` → `PropertyExport.cpp:347-408`, and the `FWeakObjectProperty` branch that `#2` says is missing has since been added, at `:399-406`. `#4`: numeric encoders `474-500` → `:662-688` (FInt8 `:662`, FInt16 `:667`, FUInt16 `:672`, FUInt32 `:677`, FUInt64 `:682`); Weak/Lazy dispatch `604-619` → `:804-819`; `FOptionalProperty` handler `626+` → `:821-845`, the cast at `:826`. **Two corrections to `#4` found while re-deriving.** (i) Its guard macro `EDITORAUTOMATIONRPCGATEWAY_HAS_OPTIONAL_PROPERTY` no longer exists under that name — it is now `PINWRIGHT_HAS_OPTIONAL_PROPERTY`, defined `PropertyExport.cpp:26`/`:28` and used at `:821`, so the old spelling greps to nothing. (ii) Its `CPF_Transient | CPF_DuplicateTransient` skip "at the `BuildClassPropertyJson` walk site (lines 2104-2111)" is no longer inline at the walk site: it was factored into a named predicate `ShouldEmitClassDumpProperty` (`PropertyExport.cpp:1407-1435`, declared `PropertyExport.h:34` as the single source of truth for the filter), whose flag mask sits at `:1428-1432` and which the walk loop calls at `:1462`; `BuildClassPropertyJson` itself now begins at `:1437`. The mask has also **grown beyond the two flags `#4` names** — it is `CPF_Transient | CPF_DuplicateTransient | CPF_Deprecated | CPF_SkipSerialization` — so anyone reasoning from `#4` about what the dump drops is working from an understatement. Third correction: `#4` says *"All 9 tests"* in `TestAssetDumpNumericAndSmartPointerProperties.cpp`; the file carries **10** `IMPLEMENT_SIMPLE_AUTOMATION_TEST` macros and now lives at `Source/PinWright/Private/Tests/Utility/TestAssetDumpNumericAndSmartPointerProperties.cpp`. Neighbouring non-`PropertyUtils` citations from `#2` re-verified in the same pass: `WidgetXmlExporter::CollectOverriddenAttributes` `WidgetXmlExporter.cpp:236` → `Handlers/UI/WidgetXmlExporter.cpp:262`, with its `ExportPropertyToJsonValue` call at `:295` and `JsonValueToAttrString` at `:298`; `WidgetXmlHelpers::JsonValueToAttrString` `WidgetXmlUtils.h:~148-159` → `Handlers/UI/WidgetXmlUtils.h:123`, its `EJson::Object` flattening case at `:151-162`.
- `#7-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
