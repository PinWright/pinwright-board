---
id: E-asset-dump-content-aware-writes
title: "asset.dump rewrites every sidecar instead of updating content-aware"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [asset-dump, writer, determinism, io, mirror]
---

# asset.dump rewrites every sidecar instead of updating content-aware

Every direct `asset.dump` and every folder-sweep cache miss rebuilds the asset's
whole mirror directory even when only one aspect changed. `WriteAssetDump`
unconditionally deletes and recreates the directory, then writes every text file
through a temporary file (`Source/PinWright/Private/Utils/AssetDumpWriter.cpp:163-195`).
Binary sidecars are written in a second pass only after that purge
(`Handlers/Asset/AssetDumpHandler.cpp:1582-1616`). Delete/mkdir results are not
checked, and the last complete baseline is removed before any replacement write
has succeeded.

This makes an isolated aspect change rewrite all sibling sidecars and their
mtimes, multiplies disk I/O on large mirror refreshes, and turns one file-write
failure into a partial baseline instead of preserving the previous complete
dump. The `/App` refresh in the reporting session covered 8,192 assets; cache
hits avoided the path, but every cache miss and every explicit single-asset dump
still takes this rewrite-all path.

**Workaround:** rely on `.dumpcache.json` to avoid whole-asset cache misses; there
is no way to avoid sibling rewrites for a direct dump or a genuinely stale asset.
**Fix:** make the baseline write content-aware and transactional: compare exact
UTF-8/binary bytes, write only changed files, retain the previous baseline if any
planned write fails, then prune stale owned sidecars after all writes succeed.
Unify text and binary outputs under one planned manifest. Keep cache bookkeeping
based on the full present-file manifest (not only physically rewritten paths),
and expose rewritten vs unchanged counts separately. Tests should prove that an
identical write preserves mtimes, a one-aspect change preserves sibling mtimes,
stale sidecars disappear only after success, and an injected write failure leaves
the old complete baseline intact.

## History
- `#1-app-refresh-rewrite-all` `OPEN` reporter — Source-validated during the 8,192-asset `/App` dump review: `WriteAssetDump` always calls recursive `DeleteDirectory` + `MakeDirectory` and rewrites every `FDumpFile`; `DumpSingleAsset` then rewrites binary files separately. A single changed/noisy aspect therefore churns every sibling sidecar for that asset, while a failed replacement can leave only a partial baseline.
- `#2-transactional-content-writes` `IN-REVIEW` developer — Unified text and binary outputs under one manifest, compare exact bytes before writing, preserve unchanged mtimes, stage changed files before commit, retain `.dumpcache.json`, prune stale sidecars only after success, and preserve the backup directory when rollback cannot fully restore. A second cold forced `/App` sweep and a cache-hit sweep left the complete 8,689-file diff hash unchanged.
