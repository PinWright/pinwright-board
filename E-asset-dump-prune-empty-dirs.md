---
id: E-asset-dump-prune-empty-dirs
title: "asset.dump_folder reconciliation leaves empty intermediate dirs"
status: DONE
severity: Low
category: ergonomic
tags: [asset-dump, reconciliation, prune, cleanup]
---

# asset.dump_folder reconciliation leaves empty intermediate dirs

After an `asset.dump_folder` sweep with `includeLevels=false` (the default),
the dump mirror under `.editor-automation/asset-dumps/` retains empty
intermediate directories that mirror UE folders whose every asset was
filtered out (levels, `MapBuildDataRegistry`, `Maps/_GENERATED` HLOD /
Lightmass). They contain no `meta.json` anywhere below them, so the
current reconciliation pass (which is driven by stale `meta.json` records)
never sees them and never deletes them.

Observed on the latest sweep of `C:/Unity/unreal-fpv/`: **20 empty
intermediate directories** (2 under `App/`, 18 under `Game/`):

```
App/Maps/_GENERATED/20928
App/PathTracer/Showcase/Maps
Game/App/UI/Test
Game/AutomotiveMaterials/Maps
Game/CarConfigurator/Configurator/Maps
Game/DETAILED_MVillage/Scenes
Game/Hangar/Maps
Game/Levels/ChemicalPlant
Game/Levels/Factory
Game/Lighting/Maps
Game/Lighting/Schema
Game/Maps/SOUND_MAP
Game/Maps/_GENERATED/20928
Game/ObjectiveManager/Demo/Maps
Game/Railway_turntable/Maps
Game/Realistic_Starter_VFX_Pack_Vol2/Maps
Game/Scene_RoadsideConstruction/Maps/_GENERATED/20928
Game/StarterContent/Maps
Game/USCS/DEMO/ThirdPerson/Maps
Game/USCS/Maps
```

## Categorization

- **(a) Expected scaffolding — filter-skipped folders (19/20):** Source
  UE folder contained only filtered-out asset kinds and therefore wrote
  nothing into the mirror. Two distinct filter paths in
  `ShouldSkipFolderDumpAsset` (`AssetDumpHandler.cpp:1299`):
  - Level-only folders (15): contain only `.umap` + `_BuiltData.uasset`
    (`MapBuildDataRegistry`), both rejected by the
    `bIncludeLevels=false` branch (lines 1327–1348). Examples:
    `Game/Levels/ChemicalPlant` (`Chemical_wp.umap`,
    `Chemical_wp_BuiltData.uasset`, `Map_ChemicalPlant_3_wp.umap`),
    `Game/Lighting/Schema` (`Universal.umap`, `Universal_BuiltData.uasset`),
    `Game/USCS/Maps` (10× tutorial `.umap` + `_BuiltData.uasset`), etc.
  - `Maps/_GENERATED/*` HLOD/Lightmass auto-generated assets (4):
    rejected by the `HasAdjacentSegments("Maps","_GENERATED")` check
    (line 1322), e.g. `App/Maps/_GENERATED/20928`
    (`LightmassImportanceVolume_0Mesh_BC9F3DE5.uasset`),
    `Game/Maps/_GENERATED/20928` (3× `SM_StadiumRoof_*` HLOD proxies),
    `Game/Scene_RoadsideConstruction/Maps/_GENERATED/20928`
    (`B_PioneerBasicLobby_Submesh_B924093F.uasset`).
- **(b) Silent dump failures (0/20):** None. Spot-checked all 20 source
  folders; every non-empty source folder's contents matched a known
  filter rule. No `ASSET_LOAD_FAILED` ghosts.
- **(c) Orphans from prior sweeps (1/20):** `Game/App/UI/Test` is empty
  in source UE as well (project test scratch folder, currently empty);
  whether it was left behind by an earlier sweep that had content or
  scaffolded then never populated, it is harmless either way.

## Impact

Cosmetic only. The empty dirs:
- never affect correctness — readers iterate `meta.json` files, not dirs;
- don't grow unbounded — they correspond 1:1 to filtered UE folders;
- do clutter `ls`/`tree` output and add ~20 stat calls to traversal.

## Fix

Extend `ReconcileMirrorSubtree` (`AssetDumpHandler.cpp:1259`) with a
second pass after the `meta.json`-driven prune: post-order walk of
`SweptRoot`, deleting any directory that becomes empty (no files, no
non-empty subdirectories). Skip `SweptRoot` itself (preserve invariant
that the sweep root remains, mirroring the strict-subpath guard already
present at line 1279). Track deletions in the existing `DeletedPaths`
return so the test harness and logging stay symmetrical with the
asset-level prune.

Low priority — purely cosmetic. Pick up alongside any other touch to
`ReconcileMirrorSubtree`.

## History
- `#1-empty-intermediate-dirs` `OPEN` reporter — 20 empty intermediate dirs (2 App, 18 Game) found under `.editor-automation/asset-dumps/` after the latest sweep. Categorization: 19 are expected scaffolding from `ShouldSkipFolderDumpAsset` (15 level-only, 4 `Maps/_GENERATED` HLOD/Lightmass), 1 (`Game/App/UI/Test`) is also empty in source UE. No silent dump failures found. Fix proposal: add a second post-order pass to `ReconcileMirrorSubtree` that deletes directories which became empty after the asset-level prune, preserving `SweptRoot` itself.
- `#2-prune-empty-intermediate-dirs` `IN-REVIEW` developer — Added a post-order empty-dir prune pass at the end of `ReconcileMirrorSubtree`. Recursive lambda enumerates immediate subdirs via `IFileManager::FindFiles(..., Files=false, Directories=true)`, recurses first, then attempts `DeleteDirectory(..., RequireExists=false, Tree=false)` — harmless failure on non-empty dirs replaces an explicit emptiness check. `SweptRoot` itself preserved via early-return. Successful deletions appended to `DeletedPaths`. Regression test `TestAssetDumpHandler.cpp::FAssetDumpHandlerFolderReconcilePrunesEmptyDirsTest` builds `SweptRoot/{Empty/Nested, HasFile/note.txt, Live/Asset1/meta.json}`, calls production `ReconcileMirrorSubtree`, asserts `Empty/Nested` + `Empty` deleted, `SweptRoot`/`HasFile`/`LiveDir` survive, and `DeletedPaths` includes both empties. Counterfactual: reverting the post-order pass leaves the empty dirs on disk; deletion assertions fail.
- `#3-verify-fix` `DONE` tester — Verified: ran `asset.dump_folder` on `/Game/Levels` (recursive, includeLevels=false, 9 assets, completed). Pre-sweep, both `Game/Levels/ChemicalPlant` and `Game/Levels/Factory` existed as empty intermediate dirs (from ticket list). Post-sweep both are gone; non-empty siblings `Blueprints`, `Meadow`, `Salt` survived; `SweptRoot` (`Game/Levels`) itself preserved as expected.
