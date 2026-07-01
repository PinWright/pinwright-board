---
id: F-enable-world-partition-impossible
title: "No working path to enable/convert World Partition: enable_world_partition hard-refuses and create_level's bCreateWorldPartition is a no-op, gating the whole level.structure WP namespace"
status: IN-REVIEW
severity: High
category: feature
tags: [level-structure, world-partition, enable, convert, create-level, capability-gap, data-layer, hlod, grid]
---

# There is no programmatic way to make a level a World Partition level

The `level.structure` namespace advertises a full World-Partition authoring
surface — `configure_grid_size`, `create_data_layer`, `configure_hlod_layer`,
`configure_level_bounds`, `create_minimap_volume`, `assign_actor_to_data_layer`
— and the wiki spells out the intended flow:

> "The typical flow is `call("level.create", …)` first, then
> `call("level.structure.enable_world_partition", …)` and the data-layer / HLOD
> / streaming setup." (`docs/wiki/level.structure.md`)

But **every documented entry point to actually turn World Partition on is a
dead end**, so `worldPartitionEnabled` can never become `true` for any level
created or loaded through the MCP, and every WP-gated verb above is permanently
blocked. There is **no sibling RPC** that performs the enable/convert (verified
against the full `level.structure` method index — see below).

## The two dead ends

1. **`level.structure.create_level {bCreateWorldPartition:true}` is a no-op
   stub.** The handler reads `WorldSettings` then hardcodes the result to false
   (`LevelStructureHandler.cpp:240-247`):

   ```cpp
   if (bCreateWorldPartition)
   {
       AWorldSettings* WorldSettings = NewWorld->GetWorldSettings();
       if (WorldSettings)
       {
           bWorldPartitionActuallyEnabled = false; // Requires editor UI to fully enable
       }
   }
   ```

   No `UWorldPartition` is ever attached. The response is honest about it
   (`worldPartitionEnabled:false, worldPartitionRequested:true` +
   `worldPartitionNote:"World Partition must be enabled via editor UI or project
   settings for new levels"`), so this is a disclosed *missing capability*, not a
   silent lie — but the documented `bCreateWorldPartition` parameter delivers
   nothing.

2. **`level.structure.enable_world_partition {bEnableWorldPartition:true}` hard-
   refuses.** When the active world has no `UWorldPartition`, the handler returns
   a clean error instead of converting (`LevelStructureHandler.cpp:759-764`):

   ```cpp
   if (bEnable && !WorldPartition)
   {
       ResponseJson->SetStringField(TEXT("note"), TEXT("World Partition must be enabled when creating the level. Convert existing level via Edit > Convert Level"));
       Ctx.SendError(TEXT("OPERATION_FAILED"),
           TEXT("Cannot enable World Partition programmatically. Use 'Edit > Convert Level' in editor or create a new level with World Partition enabled."));
       return true;
   }
   ```

   It only ever *reports* the existing partitioned state; the documented
   "Enabling rewrites the level into a partitioned world; non-trivial migration"
   behavior is never implemented. So neither path (create-with-flag nor
   enable-on-existing) can ever produce a partitioned world via the MCP.

## Impact (the whole namespace is gated)

Because nothing can set `worldPartitionEnabled:true`, every downstream WP verb
fails its precondition by design:

- `configure_grid_size` → `[OPERATION_FAILED] World Partition is not enabled for this level` (`:802`)
- `create_data_layer` → `[WORLD_PARTITION_NOT_ENABLED] World Partition is not enabled for this level. Data layers require World Partition.`
- `configure_hlod_layer`, `configure_level_bounds`, `create_minimap_volume`,
  `assign_actor_to_data_layer` are all WP-gated the same way.

A common, reasonable task — "create an open-world map as a World Partition
level, set the streaming grid, add data layers + an HLOD layer + a minimap
volume" — is therefore **impossible end-to-end through the MCP**, even though
the namespace is explicitly built to author exactly that.

## What it should do

Implement a real programmatic enable/convert path so `worldPartitionEnabled`
can reach `true` through the MCP. Either:

- Make `level.structure.enable_world_partition` actually perform the conversion
  on the active world (the engine exposes this: `UWorldPartitionConvertCommandlet`
  / `FWorldPartitionEditorModule::ConvertMap`, and `UWorld::CreateOrRepairWorldPartition`
  for a fresh world), rewriting the level into a partitioned world as the wiki
  already promises; and/or
- Make `level.structure.create_level {bCreateWorldPartition:true}` attach a real
  `UWorldPartition` at creation (e.g. `GEditor->NewMap(/*bIsPartitionedWorld=*/true)`
  or `UWorld::CreateOrRepairWorldPartition`) instead of the hardcoded
  `bWorldPartitionActuallyEnabled = false` stub.

Until one of these exists, the `bCreateWorldPartition` param and the entire
WP-gated `level.structure` surface are non-functional. (This is the missing
*capability*; the false `saved:true`/no-`.umap` persistence defect is tracked
separately in `B-create-level-saved-true-no-umap`, and the `performance.configure_world_partition`
CVar-echo defect in `B-configure-world-partition-silent-noop` — both distinct
from this enablement gap.)

## Repro (replayed via mcp__editor-automation__call)

1. `level.structure.create_level {levelName:"OracleWPGapProbe_x7", levelPath:"/Game/Maps", bCreateWorldPartition:true, save:false}`
   → `{existsAfter:true, assetClass:"World", worldPartitionEnabled:false,
   worldPartitionRequested:true, worldPartitionNote:"World Partition must be
   enabled via editor UI or project settings for new levels"}`. The requested WP
   flag produced a non-WP world.
2. `level.structure.enable_world_partition {bEnableWorldPartition:true}` (active
   world is non-WP) → `[OPERATION_FAILED] Cannot enable World Partition
   programmatically. Use 'Edit > Convert Level' in editor or create a new level
   with World Partition enabled.`
3. `level.structure.get_level_structure_info {}` →
   `{worldPartitionEnabled:false, hlodLayers:[]}` — confirms the world is still
   non-partitioned; every WP-gated verb (`configure_grid_size`,
   `create_data_layer`, …) then fails its `World Partition is not enabled`
   precondition.
4. Namespace check: the full `level.structure` method index (`docs/wiki/level.structure.md`)
   contains no other verb that enables/converts WP — `enable_world_partition` is
   the sole intended toggle, and it refuses.

Source: `LevelStructureHandler.cpp:240-247` (create stub) and `:759-764`
(enable hard-refuse).

## History
- `#2-create-attaches-wp` `IN-REVIEW` developer — Implemented the create-with-flag enable path (one of the two fixes the ticket proposes). `level.structure.create_level {bCreateWorldPartition:true}` no longer hardcodes `bWorldPartitionActuallyEnabled = false`: it now routes world creation through the engine's own `UWorld::CreateWorld(..., &IVS)` with `FWorldInitializationValues::CreateWorldPartition(true)`, which runs `InitializeNewWorld`'s canonical WP setup (SetUseActorFolders + ConvertAllActorsToPackaging + `UWorldPartition::CreateOrRepairWorldPartition`) and attaches a real `UWorldPartition`. The response field is now derived from `NewWorld->GetWorldPartition() != nullptr` (honest, not a constant), so `worldPartitionEnabled` reaches `true` and the whole WP-gated `level.structure` namespace (`configure_grid_size` / `create_data_layer` / `configure_hlod_layer` / `configure_level_bounds` / `create_minimap_volume` / `assign_actor_to_data_layer`) becomes reachable on the created world. The residual `worldPartitionNote` now fires only on the unexpected engine-declined-to-attach failure path. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Level/LevelStructureHandler.cpp` (added `#include "Engine/WorldInitializationValues.h"`; rewrote the create/IVS/WP-state block at the old `:224`/`:240-247` lines and the note at `:343`). Regression test: `EditorAutomationRpcGateway.level.structure.create_level.AttachesWorldPartition` in `Source/EditorAutomationRpcGateway/Private/Tests/World/TestLevelHandlers.cpp` — invokes the production handler with `{bCreateWorldPartition:true, save:false}` against a throwaway GUID path and asserts (1) the response reports `worldPartitionEnabled:true` (fails against the reverted hardcoded-false stub) and (2) the actual created `UWorld` returns true for `IsPartitionedWorld()` / non-null `GetWorldPartition()` (fails if the IVS is dropped); throwaway world discarded via the existing `DiscardProbeMapPackage` helper. NOTE for review: this fixes the create-with-flag dead end and unblocks the namespace end-to-end (create WP level → configure grid / data layers / HLOD). The second proposed path — making `enable_world_partition` convert an already-loaded non-WP world in place — is deliberately NOT changed here: in-process conversion of a live initialized world requires the save+`IWorldPartitionEditorModule::ConvertMap`+reopen commandlet round-trip (heavyweight, reopens the map) and is higher-risk; the create path is the supported, low-risk unblock. `enable_world_partition` still hard-refuses on a non-WP active world (its error already points users to create-with-flag, which now works).
- `#1-initial-repro` `OPEN` reporter — REALISM-mode attempt to build a World Partition "OpenWorldProto" map failed at the very first WP step. Replayed against the live editor: `level.structure.create_level {bCreateWorldPartition:true, save:false}` returned `worldPartitionEnabled:false` (no `UWorldPartition` attached — hardcoded stub at `LevelStructureHandler.cpp:240-247`), and `level.structure.enable_world_partition {bEnableWorldPartition:true}` on the active non-WP world returned `[OPERATION_FAILED] Cannot enable World Partition programmatically` (`:759-764`). Confirmed from the `level.structure` wiki index that `enable_world_partition` is the only WP-enable verb in the namespace and there is no sibling that converts — so `worldPartitionEnabled` can never become true via MCP, gating `configure_grid_size` / `create_data_layer` / `configure_hlod_layer` / `configure_level_bounds` / `create_minimap_volume` (all return `World Partition is not enabled`). The wiki's documented "typical flow" (`level.create` → `enable_world_partition` → data-layer/HLOD setup) is therefore impossible end-to-end. Dedup: distinct from `B-create-level-saved-true-no-umap` (false `saved:true`/no `.umap` persistence — that ticket explicitly scopes the WP-not-enabled field OUT as a disclosed limitation), `B-level-create-makes-wp-map` (`level.create` wrongly building a WP map — opposite direction, different handler), and `B-configure-world-partition-silent-noop` / `E-perf-wp-configure-readback-thin` (`performance.configure_world_partition` CVar echo — different method). No existing F- ticket references enabling/converting World Partition (ripgrep over OPEN + DONE + WONTFIX clean).
</content>
</invoke>
