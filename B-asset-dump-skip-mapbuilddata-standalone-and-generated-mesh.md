---
id: B-asset-dump-skip-mapbuilddata-standalone-and-generated-mesh
title: "asset.dump_folder dumps standalone _BuiltData packages and Maps/_GENERATED meshes despite includeLevels=false"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, levels, filter]
---

# asset.dump_folder dumps standalone _BuiltData packages and Maps/_GENERATED meshes despite includeLevels=false

`F-asset-dump-folder-skip-levels-by-default` filters MapBuildDataRegistry rows that share the level's package (PKG_ContainsMap on the level package). Standalone companion `_BuiltData` packages — the modern, default UE OFPA layout — have no `PKG_ContainsMap` flag, so they pass the filter and produce 121 useless empty-properties dumps in a 30k sweep.

Maps/_GENERATED helper meshes (LightmassImportanceVolume_*Mesh_*) similarly leak through.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Maps/Demonstration_BuiltData/` — empty 4-byte properties.json.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Maps/L_PDS_ChemicalPlant_BuiltData/`.
3. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Levels/ChemicalPlant/Chemical_wp_BuiltData/`.
4. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Maps/_GENERATED/.../LightmassImportanceVolume_0Mesh_BC9F3DE5/`.
5. Observe: dumped despite `includeLevels=false`.

**Fix (proposed):** Filter at `AssetDumpHandler.cpp:1081-1091` should also reject `Data.AssetClassPath == MapBuildDataRegistry`-class assets directly, not rely solely on `PKG_ContainsMap`. Likewise add a pattern filter for `Maps/_GENERATED/` package paths.

## History
- `#1-initial-repro` `OPEN` reporter — Standalone `_BuiltData` packages (modern UE OFPA layout, no `PKG_ContainsMap`) and `Maps/_GENERATED` helper meshes leak through the level filter despite `includeLevels=false`. Sample paths: `App/Maps/Demonstration_BuiltData/`, `Game/Maps/L_PDS_ChemicalPlant_BuiltData/`, `Game/Levels/ChemicalPlant/Chemical_wp_BuiltData/`, `Game/Maps/_GENERATED/.../LightmassImportanceVolume_0Mesh_BC9F3DE5/`. 121 useless empty dumps in a 30k sweep.
- `#2-skip-builtdata-generated` `IN-REVIEW` developer — Updated asset.dump_folder filtering to skip standalone MapBuildDataRegistry packages when includeLevels=false and skip Maps/_GENERATED helper packages before queueing. Added regression coverage in TestAssetDumpHandler.cpp for standalone _BuiltData and generated map helper mesh FAssetData rows.
- `#3-verify-fix` `DONE` tester — Verified: ran asset.dump_folder on /Game/Maps with includeLevels=false to outRoot=...asset-dumps-verify-builtdata (122 assets dumped, skipCount=0). Recursive find for *_BuiltData*, _GENERATED, *LightmassImportanceVolume*Mesh* in the dump root returned zero matches. AssetDumpHandler.cpp:1322-1342 contains the Maps/_GENERATED adjacent-segment filter and the MapBuildDataRegistryClassPath direct class filter as claimed.
