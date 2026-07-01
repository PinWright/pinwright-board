---
id: F-dump-world-metadata
title: "asset.dump writes no useful data for .umap (UWorld) assets"
status: DONE
severity: Medium
category: feature
tags: [asset-dump, umap, uworld, world-settings, level-blueprint, streaming]
---

# asset.dump writes no useful data for .umap (UWorld) assets

When `asset.dump` or `asset.dump_folder` processes a `.umap` file, the asset
resolves to a `UWorld` object. The generic else-branch fires and writes only
`meta.json` + `properties.json`, where `properties.json` is the diff between
the `UWorld` CDO and its parent — essentially empty because `UWorld` CDOs carry
no project-specific data. An AI agent inspecting a level gets nothing useful.

Three high-value pieces of level state were missing:

- **World Settings** (`world_settings.json`): The `AWorldSettings` actor holds
  gameplay-critical overrides — gravity, kill Z, game mode override, navigation,
  lightmap/reflection settings. Diffing against the `AWorldSettings` CDO
  surfaces exactly what the level author changed.

- **Level Blueprint** (`level_bp.txt`): Many levels wire streaming triggers,
  sequencer playback, or one-off initialization logic in the Level Blueprint.
  Exporting it as BPIR text makes it readable and diffable without opening the
  editor.

- **Streaming sublevels** (`sublevels.json`): The list of `ULevelStreaming`
  entries describes the level's composition — which sublevels are loaded
  initially, their streaming class, and their package names. Essential for
  understanding any large multi-sublevel map.

**Intentionally skipped:** per-actor iteration. Actors either live in World
Partition external packages (already reachable via the standard dump path as
individual assets) or are embedded in the `.umap` (too noisy — every transform
tweak would churn the dump). Neither case benefits from bulk actor iteration
here.

**Fix:** Added a `Cast<UWorld>` branch in `BuildAllFilesForAsset` (before the
generic else-branch) in `AssetDumpHandler.cpp`. Three new `DumpFileNames`
constants added to `AssetDumpHandler.h`. All three new names registered in
`LoadBaselineDumpFiles` (Canonical array) and the diff-mode `NonMetaCanonical`
list so baseline comparison and mirror reconciliation see them.

## History
- `#1-feature-request` `OPEN` reporter — UWorld dumps produce only an empty World CDO properties diff; world settings, level BP, and sublevel list are all missing
- `#2-add-world-branch` `IN-REVIEW` developer — Added `Cast<UWorld>` branch in `BuildAllFilesForAsset` (~line 166 of AssetDumpHandler.cpp); writes `world_settings.json` (AWorldSettings diff vs CDO), `level_bp.txt` (BPIR of PersistentLevel script BP, skipped if absent/empty), `sublevels.json` (streaming sublevel list, skipped if empty); three new DumpFileNames constants in AssetDumpHandler.h; Canonical and NonMetaCanonical arrays updated
- `#3-fix-ue56-api` `IN-REVIEW` developer — Compile error fix: `bInitiallyLoaded`/`bInitiallyVisible` removed in UE 5.6; switched to `ShouldBeLoaded()` and `GetShouldBeVisibleFlag()` getters; renamed JSON keys to `shouldBeLoaded`/`shouldBeVisible`.
- `#4-agent-verified-pass` `IN-REVIEW` tester — PASS. Test: `asset.dump` on `/App/App/LobbyMap/HomeScreenMap`. Output: `world_settings.json` (full AWorldSettings property metadata with overridden values + inheritance flags) and `level_bp.txt` (BPIR `# ==== Graph: EventGraph (ubergraph) ====` with empty BeginPlay/Tick) both written; `sublevels.json` correctly absent (HomeScreenMap has no streaming sublevels — skip-when-empty working).
- `#5-reverified-this-pass` `DONE` tester — Re-verified existing dump from #4 still shows the three new file kinds present where applicable. `world_settings.json` for HomeScreenMap has property metadata with `inherited_from`, `is_overridden_locally` flags. `level_bp.txt` has `# ==== Graph: EventGraph (ubergraph) ====` with empty Tick/BeginPlay handlers. `sublevels.json` correctly absent for the no-sublevel map. Skip-when-empty semantic confirmed.
