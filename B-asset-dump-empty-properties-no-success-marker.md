---
id: B-asset-dump-empty-properties-no-success-marker
title: "asset.dump empty properties.json indistinguishable from skipped/failed dump (FunctionLib/MacroLib/Interface/no-override BPs)"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, properties, ambiguous-empty]
---

# asset.dump empty properties.json indistinguishable from skipped/failed dump (FunctionLib/MacroLib/Interface/no-override BPs)

Several legitimate cases produce `{}`-only `properties.json`: BlueprintFunctionLibrary / BlueprintMacroLibrary / BlueprintInterface (no UProperties to dump), and child BPs that override no parent defaults (dumpSchemaVersion=3 design).

However the empty `{}` is indistinguishable from a skipped/failed dump. Consumers can't tell "this is intentionally empty" from "the dumper failed silently".

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Blueprints/Data/BP_FunctionLibrary/properties.json` — `{}`, but `bpir.txt` has multiple function graphs.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Blueprints/Data/ML_AC_MacroLibrary/properties.json` — `{}`, bpir has macros.
3. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Blueprints/Data/BPI_GamePlay/properties.json` — `{}`, bpir has interface methods.
4. Observe: 27 total `{}` properties.json across one slice on Blueprints with non-trivial parent state.

**Fix (revised):** Persist the properties aspect state in `meta.json` as `propertiesStatus` while keeping `properties.json` as a pure property-name map. Empty `properties.json` remains valid for no-overrides / no-UProperty Blueprint classes, but `meta.json.propertiesStatus` distinguishes `empty/no_overrides`, `empty/no_uproperties_on_blueprint_type`, and `error/generated_class_missing`. Do not duplicate function, macro, or interface graph names into `properties.json`; graph content remains in `bpir.txt`.

## History
- `#1-initial-repro` `OPEN` reporter — Empty `{}` `properties.json` is ambiguous: legitimate (FunctionLib/MacroLib/Interface, no-override child BPs) vs. failed dump are indistinguishable. Sample paths: `App/Blueprints/Data/BP_FunctionLibrary/properties.json`, `App/Blueprints/Data/ML_AC_MacroLibrary/properties.json`, `App/Blueprints/Data/BPI_GamePlay/properties.json`. 27 `{}` properties.json files in one slice on BPs with non-trivial parent state.
- `#2-properties-status-meta` `IN-REVIEW` developer — Added durable meta.json.propertiesStatus for Blueprint properties dumps while preserving properties.json as a pure property map; covered empty FunctionLibrary/MacroLibrary/Interface and null-GeneratedClass cases in TestAssetDumpBuilder.cpp.
- `#3-verify-fix` `DONE` tester — Verified: re-dumped /App/Blueprints/Data/BP_FunctionLibrary and /App/Blueprints/Data/BPI_GamePlay. Both meta.json now carry dumpSchemaVersion=5 and propertiesStatus={status:"empty", reason:"no_uproperties_on_blueprint_type", blueprintType:"FunctionLibrary"|"Interface"}; properties.json remains `{}`.
