---
id: B-asset-dump-meta-assettype-degenerate
title: "asset.dump meta.json assetType is hardcoded 'UObject' for ~92% of assets — provides no signal beyond className"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, meta, schema]
---

# asset.dump meta.json assetType is hardcoded 'UObject' for ~92% of assets — provides no signal beyond className

28,026 of 30,340 meta.json files (~92%) have `assetType: "UObject"`. Only 1767 say `Blueprint` and 547 `WidgetBlueprint`. The field is degenerate as a discriminator — consumers must read `className` to know what type they're looking at.

The matching test (`TestAssetDumpBuilder.cpp:49-53`) explicitly accepts `"UObject"` so the regression test is also too permissive.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Materials/Mirror/meta.json` — `className: "Material"`, `assetType: "UObject"`.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Textures/icon_01/meta.json` — `className: "Texture2D"`, `assetType: "UObject"`.
3. Observe: `assetType` is `"UObject"` despite the class indicating a more specific type.

**Fix (proposed):** `BuildMetaJson` only branches Blueprint/WidgetBlueprint and falls through to `AssetType = "UObject"`. Either map common engine roots to a useful taxonomy (`Mesh`/`Texture`/`Audio`/`Material`/`Niagara`/`Animation`/...) or drop the field and let `className` be the canonical discriminator.

## History
- `#1-initial-repro` `OPEN` reporter — 28,026 of 30,340 meta.json (~92%) carry `assetType: "UObject"` regardless of actual class. Sample paths: `App/Materials/Mirror/meta.json` (`className: "Material"`), `App/Textures/icon_01/meta.json` (`className: "Texture2D"`). Test `TestAssetDumpBuilder.cpp:49-53` accepts the degenerate value, so coverage doesn't catch it.
- `#2-removed-assettype-field` `IN-REVIEW` developer — Dropped `assetType` from `Utils/AssetDumpBuilder.cpp::BuildMetaJson` (the field was hardcoded "UObject" for ~92% of assets and added zero discrimination beyond `className`); bumped `dumpSchemaVersion` to 4. Updated `Tests/Private/Utility/TestAssetDumpBuilder.cpp` (`FAssetDumpBuilderMetaJsonShapeTest` asserts absence + schema version; `FAssetDumpBuilderMetaJsonBpStubClassTest` no longer references `assetType`). Removed `assetType` from `docs/wiki/asset.md` schema list.
- `#3-verify-fix` `DONE` tester — Verified: ran `asset.dump` on `/App/Textures/T_MirrorFrame01_BC.T_MirrorFrame01_BC` (Texture2D, previously a degenerate case). Resulting `meta.json` contains only `assetPath`, `className: "Texture2D"`, `dumpSchemaVersion: 5`, `parentClass`, `pluginVersion` — no `assetType` field. Schema version is now 5 (additional bump since the IN-REVIEW entry), confirming the field removal landed.
