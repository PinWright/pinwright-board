---
id: B-asset-dump-struct-array-as-export-text-strings
title: "asset.dump struct-array elements dumped as embedded ExportText strings instead of JSON objects (TextureParameterValues, etc.)"
status: DONE
severity: Low
category: bug
tags: [asset-dump, properties, struct-array]
---

# asset.dump struct-array elements dumped as embedded ExportText strings instead of JSON objects (TextureParameterValues, etc.)

A texture reference inside a struct array is stored as a JSON string containing UE's export-text format — a doubly-nested string-in-struct-in-string with embedded single quotes and escaped double quotes.

Hard for downstream consumers to follow texture references back to their dump folder without parsing UE's export-text grammar.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/PathTracer/Demo/Mannequin/Character/Materials/MI_Female_Body/properties.json`.
2. Observe: `TextureParameterValues` value is `"(ParameterInfo=...,ParameterValue=\"/Script/Engine.Texture2D'/App/.../T_Female_Mask.T_Female_Mask'\",...)"` — string-in-struct-in-string.

**Fix (proposed):** Same generic-ExportText path as cluster `tmap-struct-key-mangled`. Expand struct array elements into JSON objects (TArray-of-FStruct → array of JSON objects). Lower priority because the data IS recoverable.

## History
- `#1-initial-repro` `OPEN` reporter — `TextureParameterValues` and similar struct-array UPROPERTYs dump as escaped ExportText strings rather than JSON objects, requiring downstream parsers to handle UE's export-text grammar. Sample path: `App/PathTracer/Demo/Mannequin/Character/Materials/MI_Female_Body/properties.json`.
- `#2-struct-array-recurse-into-json-object` `IN-REVIEW` developer — Added `StructToJsonObject` helper in `PropertyUtils.cpp` and dispatched `FStructProperty` inner elements through it from the FArrayProperty branch (replacing the ExportText fallback). Regression test `FPropertyUtilsArrayStructElementExportTest` added under `EditorAutomationRpcGatewayTests/Private/Utility/`.
- `#3-verify-fix` `DONE` tester — Verified: re-ran `asset.dump` on `/App/PathTracer/Demo/Mannequin/Character/Materials/MI_Female_Body`; `TextureParameterValues` now emits an array of JSON objects with separate fields (`ParameterInfo`, `ParameterValue="/App/.../T_Female_Mask.T_Female_Mask"`, `ExpressionGUID`), no string-in-struct-in-string. `ScalarParameterValues` shows the same expansion.
