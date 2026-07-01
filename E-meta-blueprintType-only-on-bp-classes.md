---
id: E-meta-blueprintType-only-on-bp-classes
title: "meta.json blueprintType field only emitted on Blueprint classes (schema inconsistency)"
status: DONE
severity: Low
category: ergonomic
tags: [meta-json, schema]
---

# meta.json blueprintType field is missing on non-Blueprint assets

`meta.json` includes `blueprintType` only for assets with `_C`-suffixed Blueprint generated classes (~2,047 / 35,400 dumps). For non-Blueprint classes (Material, ObjectRedirector, UserDefinedStruct, native UObjects), the field is absent entirely.

Schema ambiguity: consumers can't tell if absence means "not a Blueprint" vs "field was forgotten".

## Fix

In `AssetDumpBuilder.cpp` meta-emission path, always emit `blueprintType`. Set it to JSON `null` for non-Blueprint assets. Document the convention in the asset-dump wiki.

## History
- `#2-blueprintType-null-schema` `IN-REVIEW` implementer — `meta.json.blueprintType` is now always present, using raw Blueprint type strings for Blueprint assets and JSON null for non-Blueprint assets. Bumped `dumpSchemaVersion` to 7 and `meta.json` aspect version to 3.
- `#1-blueprintType-conditional` `OPEN` reporter — minor schema inconsistency; affects diagnostic consumers that count populated fields.
- `#3-verify-fix` `DONE` tester — Verified: dumped `/Engine/EditorMaterials/MatineeCam_mat` (Material) emits `"blueprintType": null` and `dumpSchemaVersion: 7`; dumped `/App/App/UI/W_TrackLoaderUtil` (WidgetBlueprint) emits `"blueprintType": "Normal"` and `dumpSchemaVersion: 7`. Field now always present per the fix contract.
