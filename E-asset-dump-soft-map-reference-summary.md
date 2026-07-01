---
id: E-asset-dump-soft-map-reference-summary
title: "asset.dump should surface soft map references from data assets"
status: DONE
severity: Low
category: ergonomic
tags: [asset, dump, soft-reference, maps]
---

# asset.dump should surface soft map references from data assets

Data assets can encode important map relationships through soft object pointers or project-style path strings. Generic property dumps expose those values, but dependency classification and dump outputs do not summarize the relationship, so consumers must manually parse and follow strings to reconstruct experience-to-map chains.

**Workaround:** Manually inspect `properties.json` for map and level-definition path fields.

**Fix:** Add a generic `referenced_assets.json` or `map_references.json` summary for `TSoftObjectPtr<UWorld>` and project-style object-path strings, without automatically dumping or loading referenced maps by default.

## History
- `#1-live-repro` `OPEN` reporter — `DA_Meadow` `LevelDefinition` dumps `Map=/Game/Maps/L_PDS_Meadow.L_PDS_Meadow` only as a property, with no map placeholder/summary file; `asset.get_dependencies_classified` omits that soft UWorld reference. `DA_MeadowMap_Race_01` also stores `LevelDefinitionPath` as a string path, so the experience -> level definition -> map chain requires manual property parsing. Add a generic `referenced_assets.json` or `map_references.json` summary for `TSoftObjectPtr<UWorld>` and project-style object-path strings, without automatically dumping/loading referenced maps by default.
- `#2-typed-soft-uworld-pass-shipped` `IN-REVIEW` developer — Implemented map_references.json sidecar via BuildMapReferencesJson in PropertyUtils, dispatched from AssetDumpHandler::BuildAllFilesForAsset right after Meta emission. Typed-soft-UWorld pass only (filters on PropertyClass->IsChildOf(UWorld::StaticClass())); container traversal handles TArray<TSoftObjectPtr<UWorld>>. String-path heuristic deferred. Regression test TestAssetDumpMapReferences.cpp validates typed pass, container pass, and negative gate.
- `#3-verify-fix` `DONE` tester — Verified: asset.dump on /App/App/Experiences/Meadow/DA_Meadow.DA_Meadow listed map_references.json in Files; sidecar contents = {"references":[{"path":"/Game/Maps/L_PDS_Meadow.L_PDS_Meadow","property":"Map","source":"soft-uworld"}],"schemaVersion":1}.
