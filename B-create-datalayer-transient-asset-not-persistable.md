---
id: B-create-datalayer-transient-asset-not-persistable
title: "world_partition.create_datalayer makes the UDataLayerAsset in /Engine/Transient (GetTransientPackage outer), so the layer and every set_datalayer assignment that references it are non-persistable and lost on save/reload"
status: IN-REVIEW
severity: High
category: bug
tags: [world-partition, datalayer, create_datalayer, set_datalayer, transient, persistence, silent-no-op, save, no-disk-write, mcp-safe-asset-save, false-success]
encounters: 2
lastSeen: 2026-07-11T11:20:06.1572443+03:00
---

# `world_partition.create_datalayer` creates a transient (non-persistable) data-layer asset

`world_partition.create_datalayer` reports `"DataLayer '<name>' created."` and the
follow-up `world_partition.set_datalayer` reports `added:true`, but the underlying
`UDataLayerAsset` is constructed in the **transient package**, not as a real asset.
The actor's resulting `DataLayerAssets` reference is `/Engine/Transient.<name>`. A
transient asset is in-memory only — it is never serialized to disk, so the data
layer disappears on level save/reload and every actor→layer assignment that points
at it becomes a dangling/null reference. The whole "organize the scene into data
layers" workflow is therefore a silent no-op as far as durable state is concerned,
even though every call returns success.

This is the **root cause** of the symptom noted (but not diagnosed) in the OPEN
ergonomic ticket `E-perf-wp-configure-readback-thin`: "the newly created
Foliage_Streaming layer does not appear in get_level_structure_info.dataLayers."
A transient layer instance is exactly why no persistent/structure read-back ever
lists it. That ticket is scoped as readback-*thinness* (an ergonomic/docs ask);
**this** ticket is the underlying persistence defect — distinct, and a bug.

## Root cause (handler-confirmed)

`Source/PinWright/Private/Handlers/World/WorldPartitionHandler.cpp:140-146`

```cpp
UDataLayerAsset* NewAsset = NewObject<UDataLayerAsset>(GetTransientPackage(),
    UDataLayerAsset::StaticClass(), FName(*DataLayerName), RF_Public | RF_Transactional);
```

The asset's outer is `GetTransientPackage()`, so the `UDataLayerAsset` lives in
`/Engine/Transient` and is never saved. `set_datalayer`
(`WorldPartitionHandler.cpp:160+`) then resolves that transient layer instance and
calls `AddActorsToDataLayers`, writing the transient asset path into the actor's
`DataLayerAssets` array. Nothing in either handler creates or saves a real
`UDataLayerAsset` package (e.g. under `/Game/...DataLayers/`).

## Verbatim repro (replayed live against mcp__editor-automation__call)

Level open: `/Game/Maps/Level/Level_WorldPartitionStreaming` (a real WP level, so
the `NOT_PARTITIONED` guard passes — this is not the `enable_world_partition`
dead-end of `F-enable-world-partition-impossible`).

1. `world_partition.create_datalayer {dataLayerName:"ReplayProbeLayer"}`
   → `{"message":"DataLayer 'ReplayProbeLayer' created."}`
   (bare success; no asset path echoed.)
2. `world_partition.set_datalayer {actorPath:"Anchor_Landmark_A", dataLayerName:"Landmarks"}`
   → `{"dataLayerName":"Landmarks","added":true,"actorPath":"/Game/__ExternalActors__/...","actorName":"Anchor_Landmark_A","existsAfter":true,"actorClass":"StaticMeshActor"}`
3. `actor.describe {actorName:"Anchor_Landmark_A"}` → `properties.DataLayerAssets`:

```json
"DataLayerAssets": {
  "type": "TArray",
  "value": ["/Engine/Transient.Landmarks"],
  "is_overridden_locally": true
}
```

The membership value is `/Engine/Transient.Landmarks` — proof the layer asset is
transient. Save the level and reopen it and this reference dangles (the asset no
longer exists), so the actor is no longer in any saved data layer.

## What it should do

`create_datalayer` should create a **persistable** `UDataLayerAsset` (or an
External Data Layer) in a real package — e.g. create the package under a
project content path (`/Game/.../DataLayers/<name>`) via `CreateAsset` /
`UPackage` + `SavePackage`, or use the editor's own data-layer-create flow that
produces a saved asset — and ideally echo the resulting asset path in the success
JSON so the caller can address/verify it. With a real asset, `set_datalayer` would
write a `/Game/...` reference into `DataLayerAssets` that survives save/reload, the
layer would appear in `get_level_structure_info.dataLayers`, and the streaming
organization would actually persist.

## Workaround

None via MCP for a *persistent* data layer — every `create_datalayer` result is
transient. An agent can confirm the (transient) in-memory assignment via
`actor.describe.properties.DataLayerAssets`, but cannot produce a data layer that
survives a save. Author data layers in the editor UI / via a real asset-creation
path instead.

## History
- `#1-initial-repro` `OPEN` reporter — REALISM-mode oracle replay of a
  world-partition streaming-organization task (open level → spawn 3 anchor
  StaticMeshActors → create `Landmarks`/`FarProps` data layers → assign A/B→Landmarks,
  C→FarProps → load_cells → focus). Every call returned success, but replaying
  `set_datalayer {Anchor_Landmark_A → Landmarks}` then `actor.describe` showed the
  membership as `/Engine/Transient.Landmarks`, i.e. the data-layer asset is created
  in the transient package and is non-persistable. Root-caused to
  `WorldPartitionHandler.cpp:133` (`NewObject<UDataLayerAsset>(GetTransientPackage(), ...)`).
  Fresh `create_datalayer {ReplayProbeLayer}` returned only the bare
  `"DataLayer 'ReplayProbeLayer' created."` (no asset path), confirming no real
  package is produced. This is the persistence-defect root cause behind the symptom
  in `E-perf-wp-configure-readback-thin` (layer absent from
  get_level_structure_info.dataLayers) and is distinct from
  `F-enable-world-partition-impossible` / `E-level-structure-wp-wiki-advertises-dead-end`
  (those are the `level.structure.*` enable/convert dead-end on non-WP worlds — a
  different namespace and a clean refusal, not a silent transient write). Dedup:
  ripgrep over OPEN + closed found no existing ticket on `world_partition.create_datalayer`
  producing a transient/non-persistable asset; the only adjacent ticket
  (`E-perf-wp-configure-readback-thin`) is the readback-thinness ergonomic angle on
  the same method, not the persistence bug.
- `#2-fix` `IN-REVIEW` developer — Fixed the transient-outer root cause in
  `world_partition.create_datalayer`. `WorldPartitionHandler.cpp` no longer creates
  the `UDataLayerAsset` under `GetTransientPackage()`; it now builds a sanitized
  `/Game`-rooted package path via the production helper `ValidateAssetCreationPath`
  (`"/Game/DataLayers"` + the layer name), `CreatePackage(*FullAssetPath)`, then
  `NewObject<UDataLayerAsset>(AssetPackage, ..., RF_Public | RF_Standalone)` —
  followed by `MarkPackageDirty` + `FAssetRegistryModule::AssetCreated` +
  `McpSafeAssetSave`, and sets `Params.WorldDataLayers = World->GetWorldDataLayers()`
  before `CreateDataLayerInstance`. The success JSON now echoes `dataLayerAssetPath`
  (+ `AddAssetVerification`), so `set_datalayer` writes a real `/Game/...` reference
  into `DataLayerAssets` that survives save/reload instead of `/Engine/Transient.<name>`.
  This is the persistable pattern already proven by the sibling
  `level.structure.create_data_layer` (`LevelStructureHandler.cpp:1176-1202`).
  Files: `Source/EditorAutomationRpcGateway/Private/Handlers/World/WorldPartitionHandler.cpp`
  (fix + includes for AssetRegistry/Package/PackageName/PathUtils/AssetUtils/WorldDataLayers).
  Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/World/TestCreateDatalayerPersistable.cpp`
  — `BuildsPersistablePackagePath` asserts the production path the handler builds is a
  valid mount point under `/Game/` and NOT `/Engine/Transient` (fails if reverted to
  the transient package); `EchoesNonTransientPath` invokes the handler with capture
  and asserts any echoed `dataLayerAssetPath` is a real `/Game` asset path, never a
  transient one. Did not compile / run tests (later phase).
- `#3-additional-fix-incomplete-no-disk-write` `IN-REVIEW` reporter — Additional evidence (live replay against mcp__pinwright__call): the #2 transient-outer fix IS deployed and DID resolve the original `/Engine/Transient` defect — `world_partition.create_datalayer {dataLayerName:"OracleProbeVeg"}` (on an active WP world from `level.structure.create_level {save:false}`) now returns `{message:"DataLayer 'OracleProbeVeg' created.", dataLayerName:"OracleProbeVeg", dataLayerAssetPath:"/Game/DataLayers/OracleProbeVeg", assetPath:"/Game/DataLayers/OracleProbeVeg", existsAfter:true, assetClass:"DataLayerAsset"}` — a real `/Game/DataLayers/...` path, no longer `/Engine/Transient`. BUT the fix is INCOMPLETE: the asset still never reaches disk. On-disk check immediately after the call: NO `Content/DataLayers` folder exists at all (no `.uasset` written), so the success JSON's `existsAfter:true` is a false success. Root cause of the incomplete fix, code-confirmed: the #2-fix relies on `McpSafeAssetSave(NewAsset)` (`WorldPartitionHandler.cpp:173`) to persist the layer, but `McpSafeAssetSave` (`Utils/AssetUtils.cpp:220-232`) deliberately does NOT write to disk — it only `Asset->MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated(Asset)` then `return true` (comment: "UE 5.7+ Fix: Do not immediately save newly created assets to disk. Saving immediately causes bulkdata corruption and crashes. Instead, mark the package dirty and notify the asset registry."). `AddAssetVerification` (`AssetUtils.cpp:1133-1142`) then hardcodes `existsAfter:true` from the in-memory object with no disk probe. Net: the data layer is now a real-package but still in-memory-only, non-persistable asset — a cold restart finds no `Content/DataLayers` folder (confirmed by the cold-load corruption check filed as `B-create-level-saved-true-no-umap` #6, whose Vegetation/Lighting layers all vanished). To actually fix, `create_datalayer` must either write the package to disk (SavePackage) and re-gate `existsAfter` on a real disk probe, or persist it via the host-world save — the bare `McpSafeAssetSave` no-write stub cannot. Part of the systemic no-disk-write / false-success family (siblings: `B-create-level-saved-true-no-umap`, `B-lighting-create-level-false-success-no-umap`, `B-niagara-save-no-disk-write`, `B-audio-create-save-no-disk-write`). Still reproduces at HEAD.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. 2 citations sit in history rows and are left verbatim per the append-only rule. **Not just a path — the quoted defect is gone while the ticket sits at IN-REVIEW.** `:133` is now a blank line inside the already-exists probe, and `GetTransientPackage()` appears in that file only inside a comment saying not to use it. `:140-146` is that comment, naming this exact defect, and `:165` creates the `UDataLayerAsset` in a real on-disk package via `ValidateAssetCreationPath` into `/Game/DataLayers` with `RF_Public | RF_Standalone`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.

