---
id: E-create-datalayer-fixed-asset-folder
title: "world_partition.create_datalayer hard-codes /Game/DataLayers for the data-layer asset, so a caller cannot choose where it goes and tests cannot keep it under the scratch root"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [gap-analysis-2026-09-28, world-partition, datalayer, create_datalayer, asset-path, testing]
encounters: 1
lastSeen: 2026-09-29T17:16:39Z
---

# create_datalayer cannot place its data-layer asset

`world_partition.create_datalayer` (`Handlers/World/WorldPartitionHandler.cpp`) built the
`UDataLayerAsset` package with `ValidateAssetCreationPath(TEXT("/Game/DataLayers"), DataLayerName, ...)`
and declared no folder parameter. A project that keeps data layers elsewhere had to move the asset
afterwards, and `PinWright.world_partition.create_datalayer.EchoesNonTransientPath` could only create
its fixture in host content (`/Game/DataLayers/PersistenceProbeLayer`), outside the `/Game/PinWrightTests`
scratch root. The sibling `level.structure.create_data_layer` already takes `dataLayerAssetPath`
(folder, default `/Game/DataLayers`).

**Fix direction:** optional folder parameter defaulting to today's location, validated like other
create verbs.

## History
- `#1-hardcoded-datalayer-folder` `OPEN` reporter — Folder is a literal in the handler with no parameter; blocks caller placement and keeps the test fixture out of the scratch root.
- `#2-datalayerassetpath-param` `IN-REVIEW` developer — Added `RPC_PARAM_DEF("dataLayerAssetPath", "path", ..., "/Game/DataLayers")`, the sibling's name and default; the handler reads it with `Ctx.GetString(..., "/Game/DataLayers")` and passes it to the existing `ValidateAssetCreationPath` (sanitizes the folder, rejects traversal and unmounted roots; an explicit empty string is rejected), so an invalid folder returns `INVALID_PARAMS` before any package is created. Response unchanged: `dataLayerAssetPath` is the full asset path, as in the sibling. `docs/wiki-src/world_partition.md` gains a `## Data layer asset location` section. `EchoesNonTransientPath` now passes `/Game/PinWrightTests/DataLayers_<guid>` and asserts the echoed path equals `<folder>/PersistenceProbeLayer` (fails if the parameter is ignored). Not run.
