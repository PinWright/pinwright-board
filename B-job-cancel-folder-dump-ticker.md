---
id: B-job-cancel-folder-dump-ticker
title: "system.job_cancel reports cancelled:true but the folder-dump ticker keeps sweeping"
status: IN-REVIEW
severity: Medium
category: bug
tags: [jobs, asset-dump, cancel, silent-false-success]
encounters: 1
lastSeen: 2026-07-10T06:36:00Z
---

# system.job_cancel reports cancelled:true but the folder-dump ticker keeps sweeping

`system.job_cancel` on an in-progress `asset.dump_folder` job returns
`{cancelled:true}` and marks the registry ticket `cancelled` (with
`completed_at` set), but the actual work does not stop: the folder-dump ticker
(`TickFolderDump`, driven by `FAsyncFolderDumpState`) never consults the job
registry, so it keeps popping `PendingAssetPaths` and writing dumps until the
queue drains naturally.

Observed live: cancelled a 8,166-asset sweep one second after start
(registry showed `status:"cancelled"`, `completed_at` one second after
`started_at`); the dump tree kept growing at ~280 files/s for minutes
afterwards, to ~19,700 files, until the sweep finished on its own. The caller
is told the cancel succeeded and has no way to know work continues — a silent
false-success on a control operation (impact High, but cancel is a rare path,
so net Medium).

Follow-on effect: because `FAsyncFolderDumpState.bInProgress` stays true, a
new `asset.dump_folder` keeps returning `DUMP_IN_PROGRESS` long after the
caller was told the job was cancelled.

**Fix:** make the async dump loop registry-aware: either have `TickFolderDump`
check its job's registry status each tick and run `FinalizeAsyncDump` early on
`cancelled`, or have `FJobRegistry::CancelJob` invoke a cancellation hook that
clears `PendingAssetPaths` / stops the ticker. Ensure finalize still runs the
dirty-guard restore and does NOT run `ReconcileMirrorSubtree` on a partial
`LiveDumpDirs` set (a cancel-triggered reconcile would prune dirs for assets
that were still pending).

## History
- `#1-cancel-keeps-sweeping` `OPEN` reporter — Cancelled a live 8166-asset asset.dump_folder via system.job_cancel; registry flipped to cancelled but the ticker kept dumping ~280 files/s until natural completion, and new dump_folder calls stayed blocked with DUMP_IN_PROGRESS.
- `#2-registry-aware-cancel` `IN-REVIEW` developer — Made the async dump sweep registry-aware. TickFolderDump now consults the job ticket each tick and, the moment it is no longer running (i.e. cancelled), finalizes early via FinalizeAsyncDump(bReconcile=false) — so the sweep stops immediately, state resets (clearing the lingering DUMP_IN_PROGRESS), and the mirror reconcile is skipped so a cancel does NOT prune existing dumps for the still-pending assets. Wired at the sweep level, not via SetCancelCallback, so it covers both asset.dump_folder and async single-level asset.dump. File: Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.cpp. Regression tests (Source/PinWright/Private/Tests/Utility/TestAssetDumpFolderCancel.cpp): PinWright.asset.dump.AsyncFolder.CancelStopsSweep (adopted red test — pending assets are not dumped after cancel) and PinWright.asset.dump.AsyncFolder.CancelSkipsMirrorReconcile (cancel does not prune prior dumps for still-pending assets).
