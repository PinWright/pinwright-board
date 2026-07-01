---
id: B-asset-dump-tmap-struct-key-mangled
title: "asset.dump TMap with non-primitive key (FGuid/FSoftObjectPath/struct/int64) renders keys as 'key_N' placeholders"
status: DONE
severity: High
category: bug
tags: [asset-dump, properties, tmap]
---

# asset.dump TMap with non-primitive key (FGuid/FSoftObjectPath/struct/int64) renders keys as 'key_N' placeholders

`Utils/PropertyUtils.cpp::ExportPropertyToJsonValue` only special-cases TMap keys of type `FStrProperty`, `FNameProperty`, and `FIntProperty`. Every other key type — `FGuid`, `FSoftObjectPath`, custom UStructs, `int64`, byte enums — falls through to a placeholder branch:

```cpp
else
{
    KeyStr = FString::Printf(TEXT("key_%d"), i);
}
```

So a `TMap<FGuid, T>` dumps as `{"key_0": ..., "key_1": ...}` instead of `{"<32-hex-digit-guid>": ...}`. The map is no longer round-trippable — the LLM consumer can't recover the actual keys.

**Note on the original `FMaterialParameterInfo` example:** the prior wording of this ticket cited `UMaterial::ParameterOverviewExpansion` as the trigger, but UE 5.6 declares that property as `TMap<FString, bool>` (`Engine/Public/Materials/MaterialInterface.h:414`); the engine UI itself manufactures the keys as concatenated `Index + Association + GroupName` strings (`Editor/MaterialEditor/Private/SMaterialLayersFunctionsTree.cpp:1727`). The plugin's FString-key branch correctly emits those verbatim, so that asset is not a repro of this bug. The latent placeholder bug remains real for any genuine struct-key TMap.

**Repro:** Construct any `TMap<FGuid, T>` (or `TMap<FSoftObjectPath, T>`, `TMap<int64, T>`, `TMap<FCustomStruct, T>`) UPROPERTY on a UObject and call `asset.dump` (or invoke `ExportPropertyToJsonValue` directly). Observed JSON keys: `"key_0"`, `"key_1"`. Expected: canonical text per the engine's `ExportTextItem` for that key type.

**Fix:** Replace the `FString::Printf(TEXT("key_%d"), i)` placeholder at `Utils/PropertyUtils.cpp:560-563` with `KeyProp->ExportTextItem_Direct(KeyStr, KeyPtr, nullptr, nullptr, PPF_None)`. The same `ExportTextItem_Direct` fallback already terminates the array-element, map-value, and set-element branches at lines 524, 591, and 641, so the map-key branch now matches that pattern. For `FGuid` keys this yields the 32-character `EGuidFormats::Digits` form via `FGuid::ExportTextItem`; for `FSoftObjectPath` and custom structs it yields the canonical ExportText form.

## History
- `#1-initial-repro` `OPEN` reporter — `TMap<FMaterialParameterInfo, bool>` keys render as concatenated `Index(-1) + Association(2) + Name` strings (e.g. `"-12Global Base Color": true`) instead of structured `(Index=-1, Association=GlobalParameter, Name="Global Base Color")`. Sample paths: `Game/AutomotiveMaterials/Masters/M_Opaque_Master/properties.json`, `Game/Materials/HLOD/MiHLODMaterial/properties.json`, `Game/Building/Lighting/M_SkyDome/properties.json`.
- `#2-exporttext-fallback-for-non-primitive-keys` `IN-REVIEW` developer — Replaced `key_%d` placeholder in `Utils/PropertyUtils.cpp:560-563` with `KeyProp->ExportTextItem_Direct(...)` so any non-{Str,Name,Int} TMap key (FGuid, FSoftObjectPath, structs, int64, byte-enum) serializes as canonical text. Reformulated ticket body — original `FMaterialParameterInfo` example was wrong (UE 5.6 stores it as `TMap<FString,bool>`). Added `TestPropertyUtilsMapStructKey.cpp` regression test (`FPropertyUtilsMapStructKeyExportTest`).
- `#3-verify-fix` `DONE` tester — Verified: ran `asset.dump` on `/App/App/DA_PDSSettings` (its `BuildTypeToExperiences` is a TMap with byte-enum keys, the prior dump showed `key_0..key_5`). Regenerated `properties.json` now contains canonical enum-name keys `"CyberDrom"`, `"FreeDemo"`, `"FreeDemoLocal"`, `"FreeDemoOnline"`, `"Full"`, `"School"`, with no `key_N` placeholders.
