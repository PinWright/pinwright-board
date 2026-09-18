---
id: B-dump-folder-deferred-assets-time-out-en-masse
title: "asset.dump_folder deferred assets all time out at the end of a long sweep — one retry, an expired wall-clock timer, and a release step that unloads them first"
status: IN-REVIEW
severity: High
category: bug
tags: [asset-dump, dump-folder, asset-compilation, compile-timeout, long-sweep, queue-order, garbage-collection]
encounters: 1
lastSeen: 2026-09-17T00:00:00Z
---

# A still-compiling asset gets exactly one retry, at the end of the sweep, after its package has been unloaded

During a folder sweep an asset whose platform data is still compiling is not forced to
compile synchronously: `DumpSingleAsset` returns `bDeferredForCompilation` and the ticker
requeues it. Three details of that path compose into a sweep-ending failure: every
deferred asset in a sweep longer than the 120 s timeout is skipped with
`ASSET_COMPILE_TIMEOUT`, in a single burst at the end of the run, whether or not its
compilation ever had a chance to finish.

## Mechanism

1. **The retry lands at the far end of the queue.** `PendingAssets` is sorted descending
   and consumed with `Pop()`, so the *end* of the array is the front of the queue. The
   deferral branch requeued with `PendingAssets.Insert(Entry, 0)` — index 0, the far end.
   A deferred asset was therefore retried only after every other asset in the sweep, once.
2. **The timer is wall-clock from the first deferral.** `CompileWaitStartedSeconds` is
   stamped at the first deferral and never advanced. In any sweep longer than
   `AsyncCompilationTimeoutSeconds` (120 s) it has necessarily expired by the time the
   single retry at the end of the queue happens, so the retry cannot succeed even if the
   compile finished seconds after the deferral.
3. **The release step unloads the deferred packages.** Every
   `AssetDumpReleaseIntervalAssets` (200) processed assets the release step drains
   compilation and then clears `RF_Standalone` + `ResetLoaders` on **every** package in
   `SweepLoadedPackages`, deferred ones included. The end-of-sweep retry therefore reloads
   the package from disk; a fresh load reports `IsCompiling()` again, the already-expired
   timer fires immediately, and the asset is skipped in microseconds.

The three together make the skip deterministic rather than a real timeout: the
"120 seconds" never measures anything the asset did.

## Evidence

- **6485 `ASSET_COMPILE_TIMEOUT` skips in 17:22–17:29**, the closing minutes of an
  hour-long sweep — thousands of timeouts per minute, which is not a rate any real
  compilation stall can produce. The skips cluster at the end of the run because that is
  where the single end-of-queue retry happens for every asset deferred anywhere in it.
- **Release step 1 reports `tracked=2140`** against an interval of 200 assets: the
  tracked set is an order of magnitude larger than the interval, so a release step
  sweeps far more packages than the batch that triggered it — including every deferred
  package still awaiting its retry.
- Every one of those skips claimed "prior dump preserved" even for assets that had no
  prior dump directory at all, so the message was actively misleading about what the
  mirror still contained.

## Fix: bounded retry window

1. **Requeue near the pop end.** `ComputeDeferredRequeueIndex(PendingCount)` inserts at
   `Max(0, PendingCount - DeferredRequeueBacklog)` with `DeferredRequeueBacklog = 32`, so
   a deferral is retried after roughly 32 other entries instead of after the whole sweep.
   When the top 32 entries are all deferred the loop stops popping new work — intended
   backpressure, and it bounds the resident deferred set to ~32 packages.
2. **One retry per entry per tick.** The tick loop breaks out of its 8 ms budget the
   second time it hands back the same deferred entry: nothing compiles while the ticker
   runs, so spinning the backlog is pure waste. The existing
   `!bOtherWorkRemains || bCancelRequested` break is preserved.
3. **The release step keeps deferred packages loaded.**
   `FilterReleasablePackages(TrackedPackages, DeferredObjectPaths)` subtracts the packages
   behind the pending deferrals (`CompileWaitStartedSeconds` is keyed by object path,
   tracking is by package name, so the keys go through
   `FPackageName::ObjectPathToPackageName`) and the retained ones stay in
   `SweepLoadedPackages` so a later release step still frees them once they are dumped.
   The `FinishAllCompilation()` / `FlushAsyncLoading()`-before-unload ordering is
   unchanged — collecting an asset with an async cook in flight faults a worker thread.
4. **The 120 s cap becomes a no-progress cap.** The ticker samples
   `FAssetCompilingManager::Get().GetNumRemainingAssets()`; whenever the backlog drops
   below the last observed value, the wait start of every pending deferral is reset to
   now. Timing out now means "nothing finished compiling anywhere for 120 seconds",
   which is a genuine stall, instead of "this sweep has been running for two minutes".
5. **Honest skip message.** The timeout message branches on the dump-directory existence
   check that already ran below it: "prior dump preserved" when there is one, "no dump
   exists for this asset" when there is not. The error code is unchanged.

## Cross-reference

`B-dump-folder-sweep-never-gcs-ooms-editor` `#4` recorded this symptom in passing —
"both sweeps report a high `ASSET_COMPILE_TIMEOUT` skip rate (4,091/22,438 and
1,797/8,392, all 'prior dump preserved'), which is worth its own ticket" — while measuring
the release step that this ticket shows is one of the three contributing causes. This is
that ticket. The release step itself is correct and stays; it only needed an exemption for
packages with a deferral outstanding.

## History
- `#1-deferred-retry-window-bounded` `IN-REVIEW` developer — Filed and fixed in one pass; the mechanism was diagnosed from a sweep log before any edit. Changed `Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.cpp`: the deferral branch in `TickFolderDump` now requeues through the new pure helper `AssetDumpHandler::ComputeDeferredRequeueIndex` (inserts `DeferredRequeueBacklog = 32` entries from the pop end instead of at index 0) and breaks the tick loop when the same entry is deferred twice in one tick, tracked in a tick-local `TSet<FString>`; `RunFolderDumpReleaseStep` releases only `AssetDumpHandler::FilterReleasablePackages(SweepLoadedPackages, deferred object paths)` and keeps the rest tracked for a later step, with the `FinishAllCompilation()`/`FlushAsyncLoading()`-then-unload ordering untouched; the ticker samples `FAssetCompilingManager::Get().GetNumRemainingAssets()` into the new `FAsyncFolderDumpState::LastRemainingCompileCount` and resets every pending wait start when the backlog drops, turning the 120 s cap into a no-progress cap; the timeout message branches on the dump-directory existence check (unchanged error code `ASSET_COMPILE_TIMEOUT`). Both helpers are declared `PINWRIGHT_API` in `AssetDumpHandler.h` alongside the existing `HasAsyncCompilationTimedOut`/`ShouldRunDumpReleaseStep` and unit-tested in `Source/PinWright/Private/Tests/Utility/TestAssetDumpHandler.cpp` as `PinWright.asset.dump.AsyncFolderDump.DeferredRequeueIndex` (index arithmetic, backlog clamp, and an applied pop/insert round trip proving the retry comes back after Backlog-1 other entries; failure direction: index 0 on a long queue is the old behaviour) and `PinWright.asset.dump.AsyncFolderDump.ReleaseSkipsDeferredPackages` (object-path-to-package-name derivation including inner-object paths; failure direction: the deferred package must never appear in the releasable set). Wiki wording updated in `docs/wiki-src/asset.md`, `docs/wiki-src/asset-audit.md` and `docs/wiki-src/asset.dump-sidecars.md`; no aspect version bump, serialized output is unchanged. **Not covered by unit tests:** the tick-loop double-deferral break and the no-progress reset both need a live sweep with genuinely compiling assets — no fake-compiling-asset fixture was built. A tester should run a forced sweep over a subtree with uncompiled textures/meshes and confirm the `ASSET_COMPILE_TIMEOUT` count drops to near zero and that any remaining skip has a real compilation stall behind it.
