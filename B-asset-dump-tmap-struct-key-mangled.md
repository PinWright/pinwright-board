---
id: B-asset-dump-tmap-struct-key-mangled
title: "asset.dump TMap with non-primitive key (FGuid/FSoftObjectPath/struct/int64) renders keys as 'key_N' placeholders"
status: DONE
severity: High
category: bug
tags: [asset-dump, properties, tmap]
---

# asset.dump TMap with non-primitive key (FGuid/FSoftObjectPath/struct/int64) renders keys as 'key_N' placeholders

`Utils/PropertyExport.cpp::ExportPropertyToJsonValue` (`:605`, TMap branch `:981`) only special-cased TMap keys of type `FStrProperty`, `FNameProperty`, and `FIntProperty`. Every other key type — `FGuid`, `FSoftObjectPath`, custom UStructs, `int64`, byte enums — falls through to a placeholder branch:

```cpp
else
{
    KeyStr = FString::Printf(TEXT("key_%d"), i);
}
```

So a `TMap<FGuid, T>` dumps as `{"key_0": ..., "key_1": ...}` instead of `{"<32-hex-digit-guid>": ...}`. The map is no longer round-trippable — the LLM consumer can't recover the actual keys.

**Note on the original `FMaterialParameterInfo` example:** the prior wording of this ticket cited `UMaterial::ParameterOverviewExpansion` as the trigger, but UE 5.6 declares that property as `TMap<FString, bool>` (`Engine/Public/Materials/MaterialInterface.h:414`); the engine UI itself manufactures the keys as concatenated `Index + Association + GroupName` strings (`Editor/MaterialEditor/Private/SMaterialLayersFunctionsTree.cpp:1727`). The plugin's FString-key branch correctly emits those verbatim, so that asset is not a repro of this bug. The latent placeholder bug remains real for any genuine struct-key TMap.

**Repro:** Construct any `TMap<FGuid, T>` (or `TMap<FSoftObjectPath, T>`, `TMap<int64, T>`, `TMap<FCustomStruct, T>`) UPROPERTY on a UObject and call `asset.dump` (or invoke `ExportPropertyToJsonValue` directly). Observed JSON keys: `"key_0"`, `"key_1"`. Expected: canonical text per the engine's `ExportTextItem` for that key type.

**Fix:** Replace the `FString::Printf(TEXT("key_%d"), i)` placeholder — the map-key `else` branch, now at `Utils/PropertyExport.cpp:1012-1019` — with `KeyProp->ExportTextItem_Direct(KeyStr, KeyPtr, nullptr, nullptr, PPF_None)`. **Applied**: that call is live at `PropertyExport.cpp:1018`. The same `ExportTextItem_Direct` fallback already terminates the array-element, map-value, and set-element branches, now at `PropertyExport.cpp:973`, `:1047` and `:1100`, so the map-key branch matches that pattern. For `FGuid` keys this yields the 32-character `EGuidFormats::Digits` form via `FGuid::ExportTextItem`; for `FSoftObjectPath` and custom structs it yields the canonical ExportText form.

## History
- `#1-initial-repro` `OPEN` reporter — `TMap<FMaterialParameterInfo, bool>` keys render as concatenated `Index(-1) + Association(2) + Name` strings (e.g. `"-12Global Base Color": true`) instead of structured `(Index=-1, Association=GlobalParameter, Name="Global Base Color")`. Sample paths: `Game/AutomotiveMaterials/Masters/M_Opaque_Master/properties.json`, `Game/Materials/HLOD/MiHLODMaterial/properties.json`, `Game/Building/Lighting/M_SkyDome/properties.json`.
- `#2-exporttext-fallback-for-non-primitive-keys` `IN-REVIEW` developer — Replaced `key_%d` placeholder in `Utils/PropertyUtils.cpp:560-563` with `KeyProp->ExportTextItem_Direct(...)` so any non-{Str,Name,Int} TMap key (FGuid, FSoftObjectPath, structs, int64, byte-enum) serializes as canonical text. Reformulated ticket body — original `FMaterialParameterInfo` example was wrong (UE 5.6 stores it as `TMap<FString,bool>`). Added `TestPropertyUtilsMapStructKey.cpp` regression test (`FPropertyUtilsMapStructKeyExportTest`).
- `#3-verify-fix` `DONE` tester — Verified: ran `asset.dump` on `/App/App/DA_PDSSettings` (its `BuildTypeToExperiences` is a TMap with byte-enum keys, the prior dump showed `key_0..key_5`). Regenerated `properties.json` now contains canonical enum-name keys `"CyberDrom"`, `"FreeDemo"`, `"FreeDemoLocal"`, `"FreeDemoOnline"`, `"Full"`, `"School"`, with no `key_N` placeholders.
- `#4-repoint-citations-after-property-utils-split` `DONE` reporter — Citation maintenance only; **no behavioural claim changes and the status is untouched**. `Utils/PropertyUtils.cpp` was split into `PropertyExport.cpp` / `PropertyImport.cpp` / `PropertyInspection.cpp` / `PropertyDiff.cpp` (`PropertyUtils.h` survives only as a deprecated umbrella forwarder), so this ticket's `PropertyUtils.cpp` citations were unresolvable paths, not stale line numbers. **All five resolve into `PropertyExport.cpp`**, none into the other three; each was verified against the code at plugin HEAD `ef8a1f1b`. Map: `ExportPropertyToJsonValue` → `PropertyExport.cpp:605`, its TMap branch `:981`. The `key_%d` placeholder site `PropertyUtils.cpp:560-563` → the map-key `else` at `PropertyExport.cpp:1012-1019`, with the applied `KeyProp->ExportTextItem_Direct(...)` live at `:1018` — cited in the body twice and in `#2` once; the `#2` row is left verbatim per the append-only rule and maps here. The three sibling `ExportTextItem_Direct` fallbacks `lines 524, 591, and 641` → array-element `PropertyExport.cpp:973`, map-value `:1047`, set-element `:1100`. Test file `TestPropertyUtilsMapStructKey.cpp` still exists, at `Source/PinWright/Private/Tests/Utility/TestPropertyUtilsMapStructKey.cpp`; its test id is now `PinWright.utils.property_utils.MapStructKeyExport`. **Source-side finding, not fixed here:** the in-code comment at `PropertyExport.cpp:1016-1017` still reads *"Mirrors the value-side and array/set-element fallbacks at lines 524, 591, and 641"* — the pre-split numbers, now 973 / 1047 / 1100 in a different file. That is the same unresolvable-citation defect inside the source rather than on the board, and it is the second such stale reference found in this file (see `B-asset-dump-texture2d-duplicate-sidecars` `#3` on the `Texture2DDumpBuilder` comment). Both want a source commit, not a board one.
