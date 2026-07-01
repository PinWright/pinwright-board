---
id: B-asset-dump-folder-accounting-uses-duplicate-package-paths
title: "asset.dump_folder assetCount and dumped count mix registry rows with unique dump dirs"
status: DONE
severity: Medium
category: bug
tags: [asset, dump-folder, accounting, manifest]
---

# asset.dump_folder assetCount and dumped count mix registry rows with unique dump dirs

`asset.dump_folder` reports kickoff `assetCount` from registry-derived pending entries, but completion `dumped` from unique dump directories. Duplicate package names can make the progress and completion numbers disagree with the actual number of dump folders, especially for map packages with sidecar registry rows.

**Workaround:** Count `meta.json` files in the output root when exact dumped-directory count matters.

**Fix:** De-duplicate pending package paths before computing `TotalAssetCount`, or report separate `registryAssetCount`, `queuedPackageCount`, `dumpedDirCount`, `skipped`, and `failed` counters.

## History
- `#1-live-repro` `OPEN` reporter — `asset.dump_folder folderPath:"/Game/Brushify/Maps/Cliffs" recursive:false outRoot:"C:/tmp/mcp-dump-investigation-accounting" includeLevels:true` returned `assetCount:4`, but completed with `dumped:3`, and the output tree contained 3 `meta.json` files. Source cause: kickoff `assetCount` is `State.PendingAssetPaths.Num()` from registry-derived pending entries, while completion `dumped` is `State.LiveDumpDirs.Num()` from a `TSet` of unique dump directories. Duplicate package names, especially map package sidecar rows, make progress/completion accounting misleading.
- `#2-package-path-dedup` `IN-REVIEW` developer — added `TSet<FString> Seen` dedup guard in `AssetDumpHandler.cpp::StartAsyncFolderDump` at the `Pending`-building loop; duplicate package paths (e.g. map + co-resident MapBuildDataRegistry) are now dropped before `Pending` grows, so `R.AssetCount` matches the on-disk `meta.json` count. Regression test `EditorAutomationRpcGateway.asset.dump.AsyncFolderDump.DeduplicatesByPackagePath` drives `/Engine/Maps` with `bIncludeLevels=true`, drains the dump, then asserts `KickoffCount == MetaJsonCount`; counterfactual: reverting to unconditional `Pending.Add(MakePendingDumpPath(Data))` causes `R.AssetCount` to exceed `meta.json` count by the number of map packages with co-resident registry rows.
- `#3-verify-fix` `DONE` tester — Verified: re-ran original repro `asset.dump_folder folderPath:"/Game/Brushify/Maps/Cliffs" recursive:false outRoot:"C:/tmp/mcp-verify-B-asset-dump-accounting" includeLevels:true`; kickoff `assetCount:3` (was 4 pre-fix), completion `dumped:3`, on-disk `meta.json` count:3. All three counters now agree.
