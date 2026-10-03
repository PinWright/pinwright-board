---
id: F-dump-folder-yieldable-asset-phases
title: "asset.dump_folder needs a real per-asset time bound (yieldable load/build/write phases)"
status: OPEN
severity: Low
category: feature
tags: [asset-dump, performance, editor-freeze, ticker]
encounters: 1
costly: 0
lastSeen: 2026-10-03T00:00:00Z
rice: [1, 2, 1, 3]
priority: 6
---

# asset.dump_folder needs a real per-asset time bound (yieldable load/build/write phases)

Split from `B-dump-folder-unbounded-asset` (remaining slices after the first, fail-loudly slice). The first slice journals the asset a sweep enters (`<dump root>/.dump-inflight`) and makes later folder sweeps skip a package whose dump never returned (`ASSET_DUMP_STALLED`, listed in `<dump root>/dump-stalled.txt`). That stops a repeat freeze, but the first freeze still happens: `TickFolderDump` checks its 8 ms budget only between assets, and `DumpSingleAsset` (`LoadObject`, every sidecar builder, the transactional writer) runs as one synchronous game-thread call that a ticker cannot preempt.

Remaining slices, in order:
1. **Reported overrun.** When one asset exceeds a wall-clock threshold but does return, list it in the result (e.g. `slowAssets[]` with elapsed seconds and the phase that ran long), so a near-freeze is visible before it becomes a freeze.
2. **Async load phase.** Replace the synchronous `LoadObject` with an object-aware async load (`LoadPackageAsync` / `LoadAssetAsync`, evaluated on 5.3-5.8) so the load no longer blocks the ticker; the sweep waits across ticks with a bounded timeout and skips with a reason on expiry.
3. **Phased builders and writer.** Split `BuildAllFilesForAsset` and `AssetDumpWriter::WriteAssetDump` into per-aspect steps the ticker can interleave, publishing `currentPhase` per aspect. Any aspect that cannot yield needs a bounded failure path.
4. **Registry preflight.** `BuildFolderDumpCandidates` calls `IAssetRegistry::GetAssets` once synchronously; page it (per sub-path) if large trees show it as a long tick.
5. **Regression fixture.** A pathological-phase fixture asserting per-tick latency, progress, `system.job_status` and cancellation stay responsive.

Also open: the stalled list has no clear verb; a user deletes the line by hand. A successful single `asset.dump` of a listed package could drop it from the list.

## History
- `#1-split-from-unbounded-asset` `OPEN` developer — Filed as the remainder of `B-dump-folder-unbounded-asset` after its first slice (freeze journal + `ASSET_DUMP_STALLED` skip) landed.
- `#2-single-dump-journal-gap` `OPEN` reviewer — Gap in the first slice that belongs here: synchronous `asset.dump` (non-world assets; `AssetDumpHandler.cpp` about line 3616) neither writes the in-flight marker nor honours `dump-stalled.txt`. A stalled package with no prior dump has a missing mirror, so the analysis RPCs' hint (which now leads with `asset.dump <asset>`) steers an agent straight into the same freeze, and that freeze leaves no journal. Wrap the synchronous `DumpSingleAsset` call in the same marker write/delete, and decide whether a listed package is refused there too (with deleting the line as the retry, as for folder sweeps).
