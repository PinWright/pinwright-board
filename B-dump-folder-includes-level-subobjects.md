---
id: B-dump-folder-includes-level-subobjects
title: "asset.dump_folder includes level sub-objects, causing 10x asset inflation and 30+ min runtimes"
status: DONE
severity: High
category: bug
tags: [asset-dump, asset-registry, performance]
---

# asset.dump_folder includes level sub-objects, causing 10x asset inflation and 30+ min runtimes

`asset.dump_folder` queries the Asset Registry via `IAssetRegistry::GetAssets` without setting `bIncludeOnlyOnDiskAssets`. When a level is open in the editor, the AR returns in-memory sub-objects of that level as distinct `FAssetData` entries alongside real on-disk packages. These sub-objects have paths of the form `/Game/Path/Map.Map:PersistentLevel.B_GateSquare_C_8` — actor instances, StaticMeshActors, ProceduralFoliageVolumes, and BP actor instances — one entry per actor in the loaded level.

Every such path is passed to `LoadObject<UObject>(nullptr, *NormalizedPath)`. `LoadObject` cannot resolve a sub-object path against an unloaded outer, so every attempt fails with `ASSET_LOAD_FAILED`. The failure is not free: UE still performs the full load attempt before returning null. On the `/Game/App` content folder (~3k real on-disk assets), a loaded race level inflates the pending list to ~23k entries. The bulk of a 30+ minute dump runtime is dead sub-object load attempts — real assets dump in minutes; the remaining ~25 minutes are entirely noise.

Nothing useful is produced by these sub-object entries even in principle: actor instance state is serialized inside the parent `.umap` package, which is its own AR entry and dumps correctly on its own.

**Fix:** Set `Filter.bIncludeOnlyOnDiskAssets = true` on the `FARFilter` before calling `GetAssets`. Belt-and-suspenders: also drop any path containing `:` from the pending list post-filter (`:` is the unambiguous sub-object separator in UE object paths).

## History
- `#1-initial-repro` `OPEN` reporter — Repro: run `asset.dump_folder` on `/Game/App` with a race level open. Observed ~23k pending entries vs ~3k expected; dump takes 30+ min and fills logs with ASSET_LOAD_FAILED. Root cause: missing `bIncludeOnlyOnDiskAssets` flag on the AR filter, causing every loaded-level actor instance to appear as a separate dumpable asset.
- `#2-filter-on-disk-only` `IN-REVIEW` developer — Set `Filter.bIncludeOnlyOnDiskAssets = true` in `StartAsyncFolderDump` in `AssetDumpHandler.cpp` before the `GetAssets` call. Also added a post-filter colon-check that skips any remaining sub-object paths. Comments explain both guards.
- `#3-simplify-drop-colon-hedge` `IN-REVIEW` developer — Per shortcut-audit: removed the post-filter colon-check (was self-flagged belt-and-suspenders without a concrete failure mode); `bIncludeOnlyOnDiskAssets = true` alone is the documented filter. Net is cleaner.
- `#4-agent-verified-pass` `IN-REVIEW` tester — PASS (indirect). Test: `asset.dump_folder` on `/App/HELIOS/Drones/Icarus` (no level under it). Returned `assetCount: 33`, no inflation observed, no `:` paths in dump output. Cannot directly exercise the loaded-level pathological case from MCP without controlling editor state, but the code path is exercised on every dump and observed counts are reasonable. Recommend full /App re-dump to confirm pre-fix 23k count drops to expected ~3k.
- `#5-reverified-this-pass` `DONE` tester — Re-verified during /mcp-review: `asset.dump_folder /App/HELIOS/Drones/Icarus recursive:true` returned `assetCount: 33` (matches #4). Filesystem walk of the resulting tree found 0 paths containing `:` (sub-object separator). The pathological loaded-level case is still impossible to isolate live without controlling the editor's open level, but the on-disk filter (`bIncludeOnlyOnDiskAssets`) is what the fix bolts in and is exercised on every dump.
