---
id: F-asset-bulk-load-check
title: "No bulk package load-check verb: asset.validate is single-asset and asset.dump_folder only records a bare ASSET_LOAD_FAILED"
status: OPEN
severity: Medium
category: feature
tags: [asset, verification, load-check, bulk, jobs, content-cleanup]
encounters: 1
lastSeen: 2026-09-30T00:00:00Z
---

# No bulk "load every package under a path and report failures" verb

Verifying a content cleanup (deletes, redirector fixups, reparenting) needs "load all N packages under a folder, collect load errors, missing imports and linker warnings". No RPC does this:

- `asset.validate` (`Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp:2251`) takes one `assetPath`, calls `ResolveAsset(..., bLoadObject=true)` and returns `isValid=true` or `ERR_LOAD_FAILED` with the fixed message "Failed to load asset". It captures no log lines, and on an already-resident object it proves nothing about the on-disk bytes (see the comment at `:2286`).
- `asset.dump_folder` (`AssetDumpHandler.cpp:3592`) is a job-based sweep that loads assets, but it is the wrong tool for this: it writes full sidecars for every asset, skips unchanged assets via `.dumpcache.json` unless `force=true`, excludes levels by default, and reduces a failure to `AssetLoadFailed` with "Failed to load asset at path" (`AssetDumpHandler.cpp:2633-2638`, stub `skipReason` at `:2156`). It does not capture linker warnings or missing-import errors.
- `asset.reload` (`AssetManageHandler.cpp:2293`) is single-asset too, and no RPC name in the plugin matches `load_check`, `validate_folder` or similar.

**Workaround:** chunk a `python.execute` script that loads 100-150 packages per call, keep a batch counter in a file, and read `Saved/Logs` for warnings. Observed: 16 calls for 1519 packages in one session and 9 calls of 150 in another.

**Fix:** add `asset.load_check {folders|paths, recursive, includeLevels, wait:false}` on the existing folder-job plumbing (`StartAsyncFolderDump`/`FJobBindArgs` pattern, polled via `system.job_status`). Per package, `LoadPackage` it with a scoped `FOutputDevice` on `GLog`, since plugin code already has collectors like `FLiveCodingLogCollector` in `LiveCodingHandler.cpp:72`. Record warning/error lines from `LogLinker`/`LogUObjectGlobals`/`LogStreaming`. GC between batches, the same concern as `B-dump-folder-sweep-never-gcs-ooms-editor`. Return `{loaded, failed, withWarnings, failures:[{package, error, logLines[]}]}`, capped, with the full list spilled to a file.

## History
- `#1-bulk-load-check-gap` `OPEN` reporter — Session evidence: one subagent made 16 `python.execute loadcheck.py` calls (`batch 0 loaded 100 fails 0 ... remaining 1419`) to load 1519 packages; another made 9 `python.execute c2_load.py` calls of 150 packages each; both tracked progress in a batch-counter file. Verified from source: `asset.validate` is single-asset with no log capture (AssetManageHandler.cpp:2251), and `asset.dump_folder` loads but is cache-gated, skips levels, and reports only a generic `AssetLoadFailed` (AssetDumpHandler.cpp:2633-2638). No duplicate on the board; `F-asset-reload-from-disk` covers single-asset cold reload only.
