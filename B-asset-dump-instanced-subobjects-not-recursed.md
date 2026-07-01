---
id: B-asset-dump-instanced-subobjects-not-recursed
title: "asset.dump emits opaque path strings for Instanced UObject sub-properties instead of inlining their state"
status: DONE
severity: High
category: bug
tags: [asset-dump, properties, instanced-subobjects]
---

# asset.dump emits opaque path strings for Instanced UObject sub-properties instead of inlining their state

Whenever a UPROPERTY contains an inline sub-object (Instanced UObject), the dumper emits the sub-object path string instead of recursively expanding the sub-object's properties. Affects Enhanced Input modifiers/triggers, GameplayEffectComponents, GameplayCueNotify burst effects, anything Instanced.

The actual configuration data is lost from the dump — modifier swizzle axes, deadzone radii, inversion flags, GE component config — none of it survives.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Input/Mappings/IMC_Default/properties.json` — `Modifiers=("...IMC_Default.IMC_Default:InputModifierSwizzleAxis_0'")` (modifier config invisible).
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Input/Actions/IA_Move/properties.json`.
3. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/GameplayEffects/Damage/GE_Damage_Basic_Instant/properties.json` — `GEComponents: ["...:AssetTagsGameplayEffectComponent_0"]` (component config invisible).
4. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/GameplayCueNotifies/GCN_Test_Burst/properties.json`.

**Fix (proposed):** Property serializer falls back to `ExportText` for `TObjectPtr` / `Instanced` UObject sub-objects rather than recursing. When `UPROPERTY` has `CPF_PersistentInstance` or `CPF_InstancedReference`, recurse into the referenced object's properties (subject to a depth limit) and inline as a JSON object.

## History
- `#1-initial-repro` `OPEN` reporter — Instanced sub-objects (Enhanced Input modifiers/triggers, GameplayEffectComponents, GameplayCueNotify burst effects) are dumped as bare subobject path strings; their actual configuration is lost. Sample paths: `Game/Input/Mappings/IMC_Default/properties.json` (Modifiers as `:InputModifierSwizzleAxis_0` path), `Game/GameplayEffects/Damage/GE_Damage_Basic_Instant/properties.json` (GEComponents as `:AssetTagsGameplayEffectComponent_0` path), `Game/Input/Actions/IA_Move/properties.json`, `Game/GameplayCueNotifies/GCN_Test_Burst/properties.json`.
- `#2-instanced-subobject-recursion` `IN-REVIEW` developer — Top-level `FObjectProperty` and `ExportObjectRefElement`'s inner-FObjectProperty path in `PropertyUtils.cpp` now recurse via `ExpandInstancedSubobject` when the property carries `CPF_PersistentInstance | CPF_InstancedReference`, reusing T1's `StructToJsonObject` walker. Cycle and depth (max 3) tracked through a thread-local recursion context with typed `_kind=cycle` / `_kind=max_depth` markers. Regression test `FAssetDumpInstancedSubobjectRecursionTest` added.
- `#3-verify-fix` `DONE` tester — Verified: re-dumped IMC_Default and GE_Damage_Basic_Instant via `asset.dump`. IMC_Default `Modifiers` arrays now inline subobject state (e.g. `{"Order":"YXZ"}` for swizzle, `{"bX":true,"bY":true,"bZ":true}` for negate) instead of `:InputModifierSwizzleAxis_0` paths. GE_Damage_Basic_Instant `GEComponents` now expands to `{"EditorFriendlyName":"Asset Tags...","InheritableAssetTags":"(CombinedTags=...)"}` instead of an `:AssetTagsGameplayEffectComponent_0` path.
