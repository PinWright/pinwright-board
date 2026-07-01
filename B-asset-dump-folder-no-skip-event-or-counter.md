---
id: B-asset-dump-folder-no-skip-event-or-counter
title: "asset.dump_folder silently drops failed assets — assetCount - dumped delta has no per-skip event or reason"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, dump-folder, telemetry]
---

# asset.dump_folder silently drops failed assets — assetCount - dumped delta has no per-skip event or reason

`dumped` counter does not match initial `assetCount`. /Game sweep: `assetCount: 22466, dumped: 22259` (207 missing). /App: `assetCount: 8094, dumped: 8090` (4 missing).

No event in `jobs.jsonl` records the skipped paths, the reason, or whether the writer crashed/timed-out/refused. Caller can't distinguish "asset locked", "package corrupt", "dump-builder threw", or "load timed out".

**Repro:**
1. Run `asset.dump_folder` on `/Game` and `/App`.
2. Inspect `jobs.jsonl` completion events.
3. Observe: `assetCount`/`dumped` deltas (207 and 4 respectively); no `skipped[]` field, no per-asset failure events.

**Fix (proposed):** Per-asset failures during async ticker walk silently decrement progress. Append a `skipped` event per asset (path + reason) and surface a `skipped: [...]` array in the completion result. Related to but distinct from `B-asset-dump-folder-no-completion-signal` (now DONE).

## History
- `#1-initial-repro` `OPEN` reporter — `assetCount - dumped` deltas (207 missing on /Game, 4 missing on /App) with no per-asset skip events in `jobs.jsonl`. Caller cannot identify which assets failed or why. Reason categories (lock, corrupt, builder-threw, load-timeout) are all indistinguishable.
- `#2-skip-stub-meta-and-skipcount` `IN-REVIEW` developer — `Handlers/Asset/AssetDumpHandler.cpp::TickFolderDump` failure branch now writes a tiny stub `meta.json` to the failed asset's dump dir (`{ assetPath, className, skipped: true, skipReason, skipMessage }`) and bumps `State.SkipCount`. `FinalizeAsyncDump` adds `skipCount` to the completion result. Skip-stub dirs go into `LiveDumpDirs` so the reconcile sweep does not prune them; a successful re-dump on a later sweep overwrites the stub. Per-asset `jobs.jsonl` events were intentionally omitted — the on-disk stub is the authoritative skip record. Added `Tests/Private/Utility/TestAssetDumpHandler.cpp::FAssetDumpHandlerSkipStubTest`.
- `#3-review-fixes-reuse-and-perf` `IN-REVIEW` developer — Phase 4.5 cleanup: stub-write now uses `AssetDumpWriter::WriteAssetDump`; AssetRegistry module load hoisted; error codes extracted to `AssetDumpErrorCodes` constants in `AssetDumpHandler.h`.
- `#4-verify-skipcount-in-completion` `DONE` tester — Verified: `asset.dump_folder` on `/Game/Maps` (ticket `j_20260511T085220_4f34c84f`) returned completion event in `jobs.jsonl` with `{"dumped":122,"skipCount":0}` — `skipCount` field is now present in the completion result and the invariant `assetCount == dumped + skipCount` (122 == 122+0) holds. Source review of `AssetDumpHandler.cpp::TickFolderDump` (lines 917-993) confirms the skip branch increments `State.SkipCount`, writes the stub `meta.json` via `AssetDumpWriter::WriteAssetDump` with `skipped:true / skipReason / skipMessage`, uses cached `CachedAssetRegistry` (hoisted module load), and references `AssetDumpErrorCodes::PathTooLong` constant. Automation test `FAssetDumpHandlerSkipStubTest` is present in `TestAssetDumpHandler.cpp`.
