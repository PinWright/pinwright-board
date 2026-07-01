---
id: B-asset-dump-dir-naming-inconsistent
title: "asset.dump dir naming differs between asset types; writtenPaths can lie"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, dir-resolution, normalize-asset-path]
---

# asset.dump dir naming differs between asset types; writtenPaths can lie

`asset.dump` produces different dump-directory naming for different asset types when called with the same kind of input path:

- **Blueprint** input `/App/HELIOS/Drones/Icarus/Parent/Icarus_ParentBP` → files actually written to `App/HELIOS/Drones/Icarus/Parent/Icarus_ParentBP.Icarus_ParentBP/` (`Name.Name/` convention).
- **UWorld** input `/App/App/LobbyMap/HomeScreenMap` → files actually written to `App/App/LobbyMap/HomeScreenMap/` (no `.Name` suffix).

The same asset folder can end up with TWO sibling dirs for the same asset — one with the suffix (from a prior dump that normalized differently, or from `asset.dump_folder` enumerating via the asset registry) and one without — both containing partial or stale dump files. Observed in `App/App/LobbyMap/`: `HomeScreenMap/` (new files) sits next to `HomeScreenMap.HomeScreenMap/` (old files), with siblings `HomeScreenMap_v2.HomeScreenMap_v2/` … `HomeScreenMap_v6.HomeScreenMap_v6/` all using the suffix convention.

Worse: the `asset.dump` JSON response reports `writtenPaths` and `dumpDir` WITHOUT the `.Name` suffix even when the writer actually places the files in the `.Name`-suffixed dir. For Icarus_ParentBP the response said:

```json
"dumpDir": "C:/.../Icarus_ParentBP",
"writtenPaths": ["C:/.../Icarus_ParentBP/meta.json", ...]
```

But `Icarus_ParentBP/` does not exist on disk — files are at `Icarus_ParentBP.Icarus_ParentBP/`. Consumers of the response that try to read back from the reported path will get "no such file" errors.

## Root cause

The folder sweep at `AssetDumpHandler::StartAsyncFolderDump` (`AssetDumpHandler.cpp:529-534`) pushes `FAssetData::GetSoftObjectPath().ToString()` into the pending queue. On UE 5.6 that string is unconditionally `/Pkg/Name.Name` for any top-level asset (`FSoftObjectPath::ToString` + `FTopLevelAssetPath::AppendString`). `TickFolderDump` then forwards each pending entry to `DumpSingleAsset` directly, bypassing `TryResolveAssetPath` (which would call `FPackageName::ObjectPathToPackageName` and strip the suffix). The suffixed path reaches `AssetDumpWriter::ResolveDumpDir` unchanged → files land in `<root>/<…>/Name.Name/`. The single-asset entry point and the diff-baseline lookup both use the unsuffixed shape, so a follow-up `asset.dump <path> diff:true` against a folder-sweep baseline returns `ASSET_NO_BASELINE`. The "writtenPaths can lie" framing in #1 was a misread of stale on-disk siblings from earlier folder dumps; within a single call, response and writer agree.

## Impact

- Stale ghost directories accumulate on every code-path change that affects normalization.
- Diff mode (`asset.dump` with `diff=true`) can fail to find the baseline because it looks in the wrong dir.
- Mirror reconciliation (`ReconcileMirrorSubtree`) may delete the wrong dir or miss stale copies.
- Consumers that read back via `writtenPaths` get path-not-found errors for some asset types.

## Fix

One-line change: in the loop that builds `Pending` inside `StartAsyncFolderDump`, push `Data.PackageName.ToString()` (or equivalently `FPackageName::ObjectPathToPackageName(Data.GetSoftObjectPath().ToString())`) instead of the soft-object path. The folder sweep then produces the same unsuffixed `Name/` shape that single-asset dump and `docs/asset-dump.md` already document. No change to `ResolveDumpDir`, no change to the single-asset path, no change to diff-baseline lookup — the producer is the only point of divergence. The cleanup pass that merges existing `Name.Name/` siblings is out of scope here; track separately if we need it.

## History
- `#1-initial-repro` `OPEN` reporter — Observed during MCP verification of F-dump-world-metadata: `asset.dump /App/HELIOS/Drones/Icarus/Parent/Icarus_ParentBP` reported writtenPaths without `.Name` suffix but files landed in `Icarus_ParentBP.Icarus_ParentBP/`. `asset.dump /App/App/LobbyMap/HomeScreenMap` consistently used and reported `HomeScreenMap/` (no suffix), creating a stale `HomeScreenMap.HomeScreenMap/` sibling from a prior dump that had used the suffixed convention. Both inconsistencies trace to differing path normalization between code paths feeding `AssetDumpWriter::ResolveDumpDir` (a pure string transform).
- `#2-diff-mode-fails-on-folder-baseline` `OPEN` reporter — Same root cause manifests as a hard failure for `asset.dump diff:true`: when the baseline was produced by `asset.dump_folder` (which writes `<Name>.<Name>/`), a follow-up `asset.dump <path> diff:true` errors with `ASSET_NO_BASELINE: No baseline dump at '.../<Name>'` because the diff path resolver uses the unsuffixed convention. Reproduced on five HELIOS BPs (`HELIOS_BP`, `B_PioneerFPV`, `B_PioneerBasic`, `B_PioneerMini`, `B_Geoscan801`) — `asset.dump_folder /App/HELIOS recursive:true` produced `<Name>.<Name>/` baselines; immediately running `asset.dump diff:true` on any of those returned `ASSET_NO_BASELINE` for the unsuffixed sibling. Workaround: `cp -r <Name>.<Name>/ <Name>/` before invoking diff. The fix from #1 (canonicalize to `/Pkg/Name.Name` everywhere before `ResolveDumpDir`, including the diff baseline lookup) closes this symptom too.
- `#3-folder-sweep-canonicalize-to-unsuffixed` `IN-REVIEW` developer — Reshape: original ticket framed this as "writtenPaths can lie" with proposed direction of canonicalize-to-`Name.Name/`; both inverted. Single-asset path is consistent (response always matches writer); divergence is folder sweep producing `Name.Name/` because `TickFolderDump` bypasses `TryResolveAssetPath`. Fixed at producer: in `StartAsyncFolderDump`, push `Data.PackageName.ToString()` instead of `Data.GetSoftObjectPath().ToString()`. Folder sweep now matches the documented unsuffixed convention, diff-mode baselines line up. Reconcile/cleanup of existing stale `Name.Name/` siblings is deliberately out of scope. Test: `FolderSweepUnsuffixedDirShape` in TestAssetDumpHandler.cpp asserts canonicalization invariant.
- `#4-verified-fresh-folder-dump-unsuffixed` `DONE` tester — Verified: ran `asset.dump_folder /App/HELIOS/Drones/Icarus recursive:true outRoot:.../mcp-review-tmp-dump` (fresh root). Resulting directory tree under `mcp-review-tmp-dump/App/HELIOS/Drones/Icarus/` had **0** `Name.Name/`-suffixed dirs and **47** unsuffixed dirs (assets + parent path nodes combined). E.g. `Parent/Icarus_ParentBP/`, `SFXs/Icarus_SW/`, `StaticMeshes/Icarus_Body_SM/`. Producer-side canonicalization confirmed working; pre-existing `.Name.Name/` siblings under the live asset-dumps root are residue from older code paths and out of scope per #3.
