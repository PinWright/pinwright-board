---
id: B-level-structure-info-datalayers-stub
title: "level.structure.get_level_structure_info.dataLayers is hardcoded empty — the enumeration is an unimplemented stub, so a real data-layer summary is never reported"
status: IN-REVIEW
severity: High
category: bug
tags: [level-structure, get-level-structure-info, datalayer, readback, stub, always-empty, silent-no-op]
---

# `get_level_structure_info.dataLayers` always returns `[]` (enumeration never implemented)

`level.structure.get_level_structure_info` advertises that it returns a
"World Partition state, and **data layer summary**" of the active world, and its
response always carries a `dataLayers` array. But that array is **hardcoded
empty** — the data-layer enumeration is an unimplemented stub. No matter how many
data-layer instances are registered with the active partitioned world,
`dataLayers` is `[]`. The "data layer summary" the method promises is never
produced.

This is a separate defect from `create_data_layer` (which works) and from the
`world_partition.create_datalayer` transient-asset bug
(`B-create-datalayer-transient-asset-not-persistable`). The
`level.structure.create_data_layer` verb genuinely creates a persistable
`UDataLayerAsset` under `/Game/DataLayers/...` AND registers a real
`UDataLayerInstance` with the world (verified live below — the layer is
resolvable and assignable). The reason it never appears in any readback is **not**
transience and **not** a failed create — it is that `get_level_structure_info`
never iterates the data layers at all.

This is exactly the "separate tool-bug question for the judge" that the OPEN
ergonomic ticket `E-perf-wp-configure-readback-thin` explicitly deferred:
> "(Whether `get_level_structure_info` *should* list the new layer is a separate
> tool-bug question for the judge; …)"
The answer is yes, and the cause is the stub here.

## Root cause (handler-confirmed)

`Source/PinWright/Private/Handlers/Level/LevelStructureHandler.cpp:2154-2163`
(`get_level_structure_info`):

```cpp
if (WorldPartition)
{
    TArray<TSharedPtr<FJsonValue>> DataLayersArray;
    UDataLayerSubsystem* DataLayerSubsystem = World->GetSubsystem<UDataLayerSubsystem>();
    if (DataLayerSubsystem)
    {
        // Data layer enumeration would go here
    }
    InfoJson->SetArrayField(TEXT("dataLayers"), DataLayersArray);
}
```

`DataLayersArray` is created empty, the subsystem is fetched, and then the body is
a literal placeholder comment — `// Data layer enumeration would go here` — with
no code. The empty array is then written into the response. So `dataLayers` is
structurally guaranteed to be `[]` for every partitioned world.

The fix has a proven sibling pattern in the **same file**: the
`assign_actor_to_data_layer` handler (`LevelStructureHandler.cpp:1398`, `:1421`)
already enumerates registered layers with
`UDataLayerEditorSubsystem::Get()->GetAllDataLayers()` and reads
`DL->GetDataLayerShortName()` / `GetDataLayerFullName()`. Iterating that same set
into `DataLayersArray` (short name + asset path + type) closes the gap.

## Verbatim repro (replayed live against mcp__editor-automation__call)

1. `level.structure.create_level {levelName:"OracleDLReplay_Q1", levelPath:"/Game/Maps/Prototype", bCreateWorldPartition:true, save:false}`
   → `{worldPartitionEnabled:true, activeWorld:true, …}` (the created WP world is
   now the active editor world — post `E-create-level-save-false-world-not-active`'s fix).
2. `level.structure.get_level_structure_info {}` →
   `{currentLevel:"OracleDLReplay_Q1", worldPartitionEnabled:true, dataLayers:[]}`
   (baseline — no layers yet).
3. `level.structure.create_data_layer {dataLayerName:"Gameplay"}` →
   `{existsAfter:true, assetClass:"DataLayerAsset", dataLayerAssetPath:"/Game/DataLayers/Gameplay", dataLayerType:"Runtime", …}` (full success, persistable path).
4. `level.structure.get_level_structure_info {}` → `dataLayers:[]` **still empty**.
5. `level.structure.create_data_layer {dataLayerName:"Background"}` →
   `{existsAfter:true, assetClass:"DataLayerAsset", dataLayerAssetPath:"/Game/DataLayers/Background", …}` (success).
6. `level.structure.get_level_structure_info {}` → `dataLayers:[]` **still empty
   after two created layers**.
7. Proof the layers really exist (so the readback, not the create, is broken):
   `actor.spawn {classPath:"/Script/Engine.StaticMeshActor", actorName:"OracleDLMesh"}` → `existsAfter:true`;
   `level.structure.assign_actor_to_data_layer {actorName:"OracleDLMesh", dataLayerName:"Background"}` →
   `{dataLayerName:"Background", assigned:true, existsAfter:true}`. The `Background`
   layer is resolvable and the assignment succeeds — it is a real registered
   `UDataLayerInstance` on the active world. Yet step 6's `dataLayers` is `[]`.

## Why it matters

"Read back the structure to confirm the data layers I created" is the natural
verify step of any World-Partition setup task, and `get_level_structure_info` is
the only structure-readback verb that purports to summarize data layers. Because
the field silently reports `[]`, an agent that follows the documented
"create then read back" pattern concludes the layers were lost — even though they
were created and registered correctly. It then chases phantom failures (e.g.
suspecting the create call, or the transient-asset bug) when the actual fault is
that the readback never enumerates anything. The `dataLayers` field is, in effect,
a silent always-empty output — it cannot distinguish "zero layers" from "N layers
present" for any world.

## Workaround

To confirm a data layer exists, do **not** trust `get_level_structure_info.dataLayers`.
Instead assign an actor to it (`assign_actor_to_data_layer` returns `assigned:true`
only if the layer resolves) or inspect the actor's `DataLayerAssets` via
`actor.describe`. There is no readback verb that lists the data layers of a world.

## Fix

Implement the enumeration at `LevelStructureHandler.cpp:2154-2163`: replace the
`// Data layer enumeration would go here` placeholder with an iteration over
`UDataLayerEditorSubsystem::Get()->GetAllDataLayers()` (the exact set the
`assign_actor_to_data_layer` handler already uses at `:1398`/`:1421`), emitting one
object per `UDataLayerInstance` (`GetDataLayerShortName()`, the underlying
`UDataLayerAsset` path, `GetType()` Runtime/Editor, initially-visible/loaded). Then
the structure readback honestly reflects the layers `create_data_layer` made, and
the verify step of the WP-setup workflow becomes answerable.

## History
- `#1-initial-repro` `OPEN` reporter — SEED-mode oracle replay of an open-world
  World-Partition setup task (create WP map → grid → two data layers
  `Gameplay`/`Background` → bounds → spawn actors → assign one to `Background` →
  read structure back). Seed was `level.structure.create_level`; the create/save
  fork is already covered by `B-create-level-saved-true-no-umap` (IN-REVIEW, its
  `#3` carries this exact seed) and `E-create-level-save-false-world-not-active`
  (IN-REVIEW). The distinct, previously-unfiled defect surfaced on a **neighbor
  method**: `level.structure.get_level_structure_info.dataLayers` is hardcoded
  empty. Replayed live: created a WP world (`save:false`, now active), created two
  data layers via `level.structure.create_data_layer` (both returned
  `existsAfter:true`, `assetClass:"DataLayerAsset"`, persistable
  `/Game/DataLayers/...` paths), and `get_level_structure_info.dataLayers` stayed
  `[]` after each. Proved the layers genuinely exist by spawning a StaticMeshActor
  and assigning it to `Background` (`assigned:true` — the layer resolves through
  `DataLayerEditorSubsystem`), so the fault is the readback, not the create.
  Root-caused to `LevelStructureHandler.cpp:2154-2163`, where the data-layer
  enumeration is an unimplemented stub (`// Data layer enumeration would go here`)
  that always writes an empty `dataLayers` array. Fix pattern already proven in the
  same file by `assign_actor_to_data_layer` (`:1398`/`:1421`, uses
  `UDataLayerEditorSubsystem::Get()->GetAllDataLayers()`). Dedup: ripgrep over
  OPEN + closed found no ticket targeting the `get_level_structure_info` empty-
  enumeration stub. Distinct from `B-create-datalayer-transient-asset-not-persistable`
  (IN-REVIEW — that is the `world_partition.create_datalayer` *transient-outer*
  persistence bug on a different verb; it attributes the empty `dataLayers` to
  transience, which does not apply to the `level.structure.create_data_layer` path
  proven persistable + registered here) and from `E-perf-wp-configure-readback-thin`
  (OPEN — readback *thinness* on `world_partition.create_datalayer` /
  `configure_texture_streaming`; it explicitly deferred "whether
  get_level_structure_info should list the layer" as a separate tool-bug question
  for the judge — answered here).
- `#2-fix` `IN-REVIEW` developer — Implemented the data-layer enumeration in
  `Source/PinWright/Private/Handlers/Level/LevelStructureHandler.cpp`
  `get_level_structure_info` (the file is now under `Source/PinWright/...`, not the
  pre-rename `Source/EditorAutomationRpcGateway/...`; stub was at `:2155-2164`).
  Replaced the `// Data layer enumeration would go here` placeholder (which fetched
  the legacy `UDataLayerSubsystem` and wrote an empty array) with iteration over
  `UDataLayerEditorSubsystem::Get()->GetAllDataLayers()` — the exact editor-time
  source `assign_actor_to_data_layer` (`:1399`/`:1422`) already resolves against.
  Each `UDataLayerInstance` now emits `{name (GetDataLayerShortName), fullName
  (GetDataLayerFullName), assetPath (GetAsset()->GetPathName), type (GetType() →
  Runtime/Editor/Unknown), initiallyVisible (IsInitiallyVisible),
  initiallyLoaded (IsInitiallyLoadedInEditor)}`. All needed headers were already
  included (`DataLayerEditorSubsystem.h`, `DataLayerInstance.h`, `DataLayerAsset.h`).
  Regression test added to
  `Source/PinWright/Private/Tests/World/TestLevelHandlers.cpp`:
  `PinWright.level.structure.get_level_structure_info.EnumeratesDataLayers` — drives
  the full production round-trip (create an active WP world via
  `create_level {bCreateWorldPartition:true, save:false}` → register a Runtime layer
  via the `create_data_layer` handler → call `get_level_structure_info`) and asserts
  the created layer appears in `dataLayers` by short name with `type:"Runtime"`.
  With the stub reverted, `dataLayers` is `[]` and the membership assertion fails.
  Placed in `TestLevelHandlers.cpp` and built on the file-local `FScopedProbeWorld`
  scaffold (which wraps the WP create_level + `DestroyWorld`-then-discard teardown and
  the `FScopedEditorWorldMapGuard` map restore); 5.4+-gated like the sibling WP tests
  (5.3 partitioned teardown crashes the engine) and skipped without `GEditor`.
  Off-by-one in the ticket's `:2154-2163` line citations confirmed (current code
  `:2155-2164`); does not affect the fix. Two fix hosts converged on this same
  enumeration independently; the merged tree carries one copy of the handler change
  and one copy of the regression test. Not compiled here (a later phase compiles + runs).
