---
id: B-asset-dump-softclass-type-trailing-space
title: "asset.dump properties.json TSoftClassPtr type strings have trailing space (cosmetic)"
status: DONE
severity: Low
category: bug
tags: [asset-dump, properties, cosmetic]
---

# asset.dump properties.json TSoftClassPtr type strings have trailing space (cosmetic)

Cosmetic — `GetCPPType` returns the typename with trailing space for SoftClass properties; the dumper passes it through verbatim. Annoying for downstream consumers that match `type === "TSoftClassPtr<X>"`.

217+ properties.json files affected.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/DefaultGameData/properties.json`.
2. Observe: `"type": "TSoftClassPtr<UGameplayEffect> "` — trailing space inside the quoted value.

**Fix:** Added `TypeStr.TrimEndInline()` in `ExportPropertyToJsonValueWithInheritance` (`PropertyUtils.cpp`) immediately after the `GetCPPType()` call. Universal trim defends against any other property kind with stray engine whitespace.

## History
- `#1-initial-repro` `OPEN` reporter — `GetCPPType` returns SoftClass typenames with a trailing space, passed through verbatim. Sample path: `Game/DefaultGameData/properties.json` — `"type": "TSoftClassPtr<UGameplayEffect> "`. 217+ properties.json files affected.
- `#2-trim-cpp-type-string` `IN-REVIEW` developer — Added universal `TypeStr.TrimEndInline()` in `ExportPropertyToJsonValueWithInheritance` (`PropertyUtils.cpp`) between `GetCPPType(nullptr, CPPF_None)` and `SetStringField("type", TypeStr)`. Root cause is engine quirk in `FSoftClassProperty::GetCPPType` returning `"TSoftClassPtr<X> "` (literal trailing space at `Runtime/CoreUObject/Private/UObject/PropertySoftClassPtr.cpp:64`). Added regression test `FAssetDumpJsonShapeSoftClassTypeNoTrailingSpaceTest` exercising `UInputSettings::DefaultPlayerInputClass` (a `TSoftClassPtr<UPlayerInput>` UPROPERTY) and asserting exact equality to `"TSoftClassPtr<UPlayerInput>"`.
- `#3-verify-fix` `DONE` tester — Verified: ran `asset.dump` on `/Game/DefaultGameData`, re-read `.editor-automation/asset-dumps/Game/DefaultGameData/properties.json`. All three `TSoftClassPtr<UGameplayEffect>` entries (lines 10, 21, 32) now read `"type": "TSoftClassPtr<UGameplayEffect>"` with no trailing space; regex `TSoftClassPtr<[^>]+> "` returns zero matches.
