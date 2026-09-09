---
id: B-dump-folder-sweep-never-gcs-ooms-editor
title: "asset.dump_folder never garbage-collects during a sweep — a /Game run (22,446 assets, force) grows the resident set until the editor is OOM-killed, twice in a row"
status: IN-REVIEW
severity: High
category: bug
tags: [asset-dump, dump-folder, memory, garbage-collection, oom, editor-crash, asset-compilation, long-sweep]
encounters: 1
costly: 1
lastSeen: 2026-09-09T07:55:00Z
---

# A full `/Game` sweep loads 22k packages and never releases one of them

`asset.dump_folder {folderPath:"/Game", force:true}` on `X:\src\unreal\unreal-fpv-dev`
(UE 5.8, plugin `fa755a4f`, 22,446 assets) killed the editor **twice**: once after
13,376 assets, once after 21,677. Both deaths are out-of-memory, not the
`B-dump-folder-unbounded-asset` hard-freeze. `/App` (8,388 assets) completes in ~3 min
on the same build, so the defect is a function of sweep length, not of any one asset.

## Measured, `X:\src\unreal\unreal-fpv-dev\Saved\Logs\PDS-backup-2026.09.09-07.55.28.log`

- **Zero GC in the whole run.** `grep -c LogGarbage` over the 48-minute log returns
  **0**; the only garbage-collection-shaped line in the file at any verbosity is one.
  Nothing on the dump path ever asks for a collect.
- **584** `LogAsyncCompilation: Display: BEWARE: AssetCompile memory estimate is greater
  than available, but we're running it ... anyway!` lines, and the budget the engine
  reports collapses across the run as resident packages eat the headroom:
  - first (`:14406`, 07:20:03) `[TextureDerivedData] RequiredMemory = 12971.67 MiB,
    MemoryLimit = 7944.15 MiB`
  - last (`:49597`, 07:54:41) `[StaticMesh] RequiredMemory = 449.85 MiB,
    **MemoryLimit = 240.82 MiB**`
  `FAssetCompilingManager`'s `GetMemoryAvailableForAssetCompilation` is therefore
  reporting an editor with essentially no free working set, and it keeps scheduling
  compilation anyway.
- **Death**: `UsedVirtual 137500061696 (128.06 GiB)`, `PeakUsedPhysical 36.10 GiB`,
  `AvailableVirtual 0.01 GiB`, then
  `Fatal error: [GenericPlatformMemory.cpp:300] Ran out of memory allocating 1024 bytes`
  (`:49612`, Windows reports the page file too small). Editor process gone; every agent
  attached to that gateway lost its session.

## Root cause (source, plugin `fa755a4f`)

The sweep's only load site is
`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Asset\AssetDumpHandler.cpp:2126`
(`UObject* Asset = LoadObject<UObject>(nullptr, *NormalizedPath);`), and
`CollectGarbage`, `ForceGarbageCollection`, `UnloadPackages` and `ResetLoaders` appear
**nowhere** in `AssetDumpHandler.cpp`, `AssetDumpCache.cpp` or `AssetDumpWriter.cpp`.
Every package the sweep touches therefore stays resident until the ticker finishes or
the process dies, and the editor's idle GC never runs because the ticker is busy.

The naive remedy does not work either: a disk-loaded asset is `RF_Standalone`, which is
in `GARBAGE_COLLECTION_KEEPFLAGS`, so a plain `CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS)`
between assets would free nothing. Releasing swept packages has to be explicit.

## Proposed fix

Every N assets, or on a working-set watermark (both settable in the `UPinWrightSettings`
"Jobs" block, `Public\PinWrightSettings.h:136-158`, next to the existing job knobs):

1. **Drain compilation first.** `FAssetCompilingManager::Get().FinishAllCompilation()`
   before any collect — GC'ing an asset with an async DDC build in flight crashes on a
   worker thread (`Plugins\PinWright\docs\lessons.md:169`, the `USoundWave`
   `FAudioCookInputs` raw-reference case).
2. **Release the swept packages**, do not just ask for a collect: the
   `ClearFlags(RF_Standalone)` + `ResetLoaders` route from
   `PackageTools.cpp:564-589`. Note that `UPackageTools::UnloadPackages` also closes
   asset editors, and closing an asset editor while anything still holds a
   `TSharedPtr<FSceneViewport>` on its preview is a `check()` abort, not a leak
   (`docs\lessons.md:194`) — so either use the flag/loader route directly or guarantee
   nothing has an editor open on a swept package.
3. **Clear the raw-pointer caches before the collect.** `GSoftWorldPropertyCache`
   (`Private\Utils\PropertyExport.cpp:1573`, `TMap<TWeakObjectPtr<UClass>, TArray<FProperty*>>`)
   and `CachedByClass` (`Private\BTIR\BTIRDecompiler.cpp:104`,
   `TMap<UClass*, TArray<FStructProperty*>>`) both cache bare `FProperty*` keyed by
   class; a collect that frees a Blueprint-generated class leaves them dangling.
4. **Trigger from the ticker** with `GEditor->ForceGarbageCollection(true)`, which is
   explicitly documented as safe from that context — it only raises a flag, consumed by
   `ConditionalCollectGarbage` after `bInTick` clears
   (`Public\Dispatch\SafePoint.h:123-127`).

Acceptance: a forced `/Game` sweep completes without the editor's resident set growing
monotonically, and `MemoryLimit` in the `AssetCompile` lines does not decay toward zero.

## Distinct from the neighbouring tickets

- `B-dump-folder-unbounded-asset` (OPEN, Low) is the **latency** defect: one synchronous
  `DumpSingleAsset` cannot be preempted, so the editor hangs. That one is about a single
  pathological asset; this one is about 22,000 healthy ones and would still kill the
  editor with a perfect per-asset bound in place.
- `E-asset-dump-folder-cache-miss-reason-opaque` and
  `B-dumpcache-fingerprint-includes-marketing-version` explain **why** 22k assets had to
  be reloaded at all (a fingerprint invalidation), not why loading them is fatal.
- `B-asset-dump-game-high-skip-ratio` (WONTFIX) is about `skipCount`.

**Workaround:** sweep `/Game` in subfolders, restarting the editor between them; or do
not sweep `/Game` at all (the committed mirror covers `/App` plus the `/Game` subtrees
already dumped). Neither prevents the growth, they just cap it per run.

## History
- `#1-two-oom-kills-on-one-game-sweep` `OPEN` reporter — Filed after two consecutive OOM editor kills on `asset.dump_folder {folderPath:"/Game", force:true}` (22,446 assets) on UE 5.8 / `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`; the run died at 13,376 assets and, after a restart, at 21,677. Evidence is the log above: **0** `LogGarbage` lines in 48 minutes, **584** `BEWARE: AssetCompile memory estimate is greater than available` lines with the reported `MemoryLimit` decaying 7944.15 MiB -> 240.82 MiB, and the fatal at `UsedVirtual 128.06 GiB`. Source-verified that the sweep has exactly one load site (`AssetDumpHandler.cpp:2126`) and no collect/unload call anywhere on the dump path, and that `RF_Standalone` makes a plain keep-flags collect a no-op for these packages. Marked `costly`: two editor kills, ~48 minutes of sweep discarded twice, and the surviving partial mirror had to be reconciled by hand. **Severity note:** the README impact class would put an editor crash at `Critical`; filed `High` instead to respect the user's standing reprioritization of asset-dump sweep severity recorded on `B-dump-folder-unbounded-asset` `#6` ("does not consider it high impact"). A fixer who disagrees should raise it rather than treat `High` as a considered ceiling.
- `#2-release-swept-packages-every-n-assets` `IN-REVIEW` developer — Added `Source/PinWright/Private/Handlers/Asset/AssetDumpSweepGc.{h,cpp}`: at every asset boundary the sweep samples `FPlatformMemory::GetStats()` into per-sweep peaks and, once `DumpedCount+SkipCount` has advanced by N or `UsedPhysical` crosses a watermark, clears the two UClass-keyed raw-`FProperty` caches, `FlushAsyncLoading()` + `FAssetCompilingManager::FinishAllCompilation()`, clears `RF_Standalone` and `ResetLoaders` on the packages the sweep itself brought into memory, then `GEditor->ForceGarbageCollection(true)` (flag only, consumed after `bInTick` clears — safe from the ticker per `Dispatch/SafePoint.h`). Already-resident, dirty, rooted and world-holding packages are never released. `AssetDumpHandler.cpp` records the sweep-loaded package set before each load and calls the new `MaybeCollectGarbageDuringSweep` at the `between_assets` boundary (never mid-asset); `AsyncFolderDumpState.h` carries `SweepLoadedPackages` / `ProcessedAtLastCollect` / `GcStats`; `AddAsyncDumpDiagnostics` publishes `gcCollections`, `gcPackagesReleased`, `peakUsedPhysicalMB`, `peakUsedVirtualMB` on progress frames and the completion result. Knobs are `UPinWrightSettings::DumpSweepCollectGarbageEveryAssets` (200) and `DumpSweepMemoryWatermarkMB` (16384) in the Jobs block; 0 disables either trigger. Cache-clear entry points added: `ClearSoftWorldPropertyCache`/`GetSoftWorldPropertyCacheNum` (`Utils/PropertyExport.{h,cpp}`) and `BTIRDecompiler::{GetBlackboardSelectorProperties,ClearBlackboardSelectorPropertyCache,GetBlackboardSelectorPropertyCacheNum}` (the previously function-local `CachedByClass` was hoisted). Wiki: `docs/wiki-src/asset.md` `asset.dump_folder` section. Regression test `PinWright.asset.dump.SweepGarbageCollect.CollectsOnIntervalAfterClearingClassCaches` (`Source/PinWright/Private/Tests/Assets/TestAssetDumpSweepGarbageCollect.cpp`) drives a real sweep over `/Engine/EngineDamageTypes` plus four injected entries with the interval forced to 2 and the collect body replaced by an observer. Counterfactual: revert `MaybeCollectGarbageDuringSweep` in `TickFolderDump` and the observed collect count is 0 against the expected `floor((dumped+skipCount)/2)`; revert `ClearClassKeyedCaches()` in `RunCollectNow` and both cache sizes read non-zero at collect time instead of 0.
- `#3-own-by-start-of-sweep-baseline-not-by-queue-entry` `IN-REVIEW` developer — Verifier finding: the `#2` ownership predicate (a `FindPackage` miss immediately before an asset's own load) classified only top-level queue entries, so every transitively loaded dependency — most of a `/Game` sweep — was exempted forever and the working set still grew monotonically. Replaced with a snapshot: `AssetDumpSweepGc::SnapshotResidentPackages` (`ForEachObjectOfClass(UPackage::StaticClass(), ...)`) runs once at both `StartAsyncFolderDump` arming paths into `FAsyncFolderDumpState::BaselineResidentPackages`, and `ReleaseSweptPackages` now enumerates live residency at collect time and releases everything absent from that baseline, keeping the dirty / rooted / `UWorld` exclusions. Eligibility is re-evaluated per collect, so the previous deferred-retry set is gone. `ReleaseSweptPackages` was also promoted out of the anonymous namespace onto `AssetDumpSweepGc` so a test can drive the real release. Second verifier finding: the regression test always installed the collect override, so the production release branch was never executed — added `PinWright.asset.dump.SweepGarbageCollect.ReleaseClearsStandaloneOutsideTheBaseline`, which snapshots residency, removes two `/Engine/EngineMaterials` probes from the baseline, roots one of them, calls `ReleaseSweptPackages` with NO override, and asserts `Released == 1`, the unrooted probe lost `RF_Standalone`, and the rooted one kept it (flags restored afterwards; no extra collect forced). Counterfactual: stub `ReleaseSweptPackages` to `return 0;` and all three assertions fail. Third finding (PATH_TOO_LONG `continue` skipping the boundary) fixed at the source instead of in the test: `MaybeCollectGarbageDuringSweep` moved to the TOP of the per-asset loop body in `TickFolderDump`, the one boundary no per-asset outcome can `continue` past, so the schedule is exactly every N processed assets; the interval test's expectation is correspondingly `(processed - 1) / 2`, since the check observes counts 0..n-1. `docs/wiki-src/asset.md` updated to describe the snapshot model.
- `#4-release-step-shipped-and-measured-on-origin` `IN-REVIEW` developer — **Second, independent implementation, and the one actually published.** `#2`/`#3` describe `Source/PinWright/Private/Handlers/Asset/AssetDumpSweepGc.{h,cpp}` and settings `DumpSweepCollectGarbageEveryAssets` / `DumpSweepMemoryWatermarkMB`; none of those files or symbols exist on `origin/master` (checked `git ls-tree -r origin/master`), so that work is still unpublished on a fix host. What is on `origin/master` is `ec36edef` ("Release swept packages during asset.dump_folder to bound memory"), which solves the same defect in `AssetDumpHandler.cpp` itself rather than in a new file: a release step gated by `AssetDumpHandler::ShouldRunDumpReleaseStep` (pure predicate, unit-tested) runs from `TickFolderDumpRelease` at the top of `TickFolderDump`, before any asset work, so no per-asset `continue` can skip it. The step drains `FAssetCompilingManager::FinishAllCompilation()` + `FlushAsyncLoading()`, clears `GSoftWorldPropertyCache` (new `ClearSoftWorldPropertyCache` in `Utils/PropertyExport.{h,cpp}`) and the hoisted BTIR blackboard cache (new `BTIRDecompiler::ClearPropertyCaches`), then `ClearFlags(RF_Standalone)` + `ResetLoaders` on the packages the sweep itself loaded — ownership recorded as a `FindPackage` miss immediately before each load into `FAsyncFolderDumpState::SweepLoadedPackages`, skipping dirty, rooted, baseline-dirty, editor-world and asset-editor-open packages — and requests `GEditor->ForceGarbageCollection(true)`. A `PostReachabilityAnalysis` callback restores `RF_Standalone` on survivors (`UPackageTools::RestoreStandaloneOnReachableObjects` idiom); the sweep parks in phase `waiting_for_release_gc` until reachability is observed and `IsIncrementalPurgePending()` clears (60 s ceiling), and every state-discarding path calls `AbortDumpReleaseGcWatch()`. Knobs: `AssetDumpReleaseIntervalAssets` (200) and `AssetDumpReleaseMemoryWatermark` (0.5 of physical RAM) in the `UPinWrightSettings` "Jobs" block, either 0 to disable, plus a `DumpReleaseMinAssetsBetweenSteps` floor of 25 so an unreachable watermark cannot degenerate into one release per asset. Progress payload gains `releaseStepCount` / `releasedPackageCount` / `workingSetBytes`. **Measured on `X:\src\unreal\unreal-fpv-dev` (UE 5.8, 63 GB box):** forced `/Game` 22,438 assets 13:32–13:48 UTC (`j_20260909T133201_7a50f87b`) and forced `/App` 8,392 assets 13:52–13:57 (`j_20260909T135204_91ec2166`) **both completed** — the pre-fix build died at 13,376 and 21,677 of 22,446. 165 release steps in `Saved/Logs/PDS.log`, 165 collected, **0 timed out**; the working set sawtooths instead of growing, e.g. `release step 65 collected after 2.6s: workingSet 29.82 -> 1.84 GiB (delta 27.98 GiB)` and `release step 32 collected after 2.5s: workingSet 24.57 -> 8.38 GiB (delta 16.19 GiB)`. Mirror accounting after the run: `/App` 8,392 dump folders for 8,392 assets (exact); `/Game` 22,430 for 22,438 — the 8 missing are all `/Game/Maps/Arena/New/SM_*`, new assets that hit `ASSET_COMPILE_TIMEOUT` with no prior dump to preserve. Regression test `PinWright.asset.dump.AsyncFolderDump.ReleaseStepTrigger` (11 assertions over both triggers, the floor, and each disable path) passes; dump suite `PinWright.AssetDump+PinWright.Asset.Dump+PinWright.Assets.` 216/221, the 5 failures pre-existing and unrelated (MetaSound registry uninitialised ×2, AnimSequence `No Movie Scene found for SequencerDataModel` ×2, one PIE-mode `AssetResolution` test). **Open item for the tester:** the two implementations overlap; whoever publishes `AssetDumpSweepGc` second must reconcile rather than land both release paths. **Also observed, not fixed here:** both sweeps report a high `ASSET_COMPILE_TIMEOUT` skip rate (4,091/22,438 and 1,797/8,392, all "prior dump preserved"), which is worth its own ticket if it is not the pre-existing `B-asset-dump-game-high-skip-ratio` behaviour.
- `#5-release-test-builds-its-own-probes` `IN-REVIEW` developer — The `#3` release test skipped on every real editor host (`reason=fixture-unavailable -- release probe is shared, dirty, rooted, or already not standalone`, `Saved/PinWright/test-runs/sprint-c2/automation.log`): its `/Engine/EngineMaterials` probes are already resident there, so their eligibility depended on what the rest of the session had done to them and `AssetDumpSweepGc::ReleaseSweptPackages` was never executed. `Source/PinWright/Private/Tests/Assets/TestAssetDumpSweepGarbageCollect.cpp` now builds the probes itself: a residency snapshot is taken BEFORE they exist, two GUID-isolated `/Game` packages are created and saved through the shared `AssetDumpMismatchedNameFixture` helper (`CreatePackage` + one `RF_Public|RF_Standalone` inner asset, `SavePackage` with validate-on-save bracketed off, registry rescan, `CleanupTestAsset` teardown), their absence from the pre-creation snapshot is asserted, and the effective baseline is a current snapshot minus the two probe names so the candidate set is exactly the probes and no unrelated editor package can be released. One probe's package is rooted, `ReleaseSweptPackages` is called with NO override and no forced collect, and the assertions are `Released == 1` (the number of unrooted probes), the released probe's asset lost `RF_Standalone`, and the rooted one kept it; `RF_Standalone` is restored before teardown. Dirty/rooted/standalone fixture preconditions are now hard assertions instead of skip reasons — the only remaining skip is a probe package that cannot be created or saved at all. Test id unchanged: `PinWright.asset.dump.SweepGarbageCollect.ReleaseClearsStandaloneOutsideTheBaseline`. Counterfactual: stub `ReleaseSweptPackages` to `return 0;` and the release count reads 0 against 1 and the probe still carries `RF_Standalone`; the rooted-probe assertion is the control and keeps passing.
- `#6-merged-onto-upstream-release-step` `IN-REVIEW` developer — On rebase onto origin/master, upstream commit ec36edef ("Release swept packages during asset.dump_folder to bound memory", from the other checkout) was found to implement this same fix more thoroughly (compilation drain, PostReachabilityAnalysis RF_Standalone restore, dirty/rooted/world/editor-open exclusions, awaited collect with a 60 s ceiling; settings AssetDumpReleaseIntervalAssets=200 and AssetDumpReleaseMemoryWatermark=0.5 of RAM; payload releaseStepCount/releasedPackageCount/workingSetBytes). The implementation described in #2..#5 (AssetDumpSweepGc.*, DumpSweep* settings, TestAssetDumpSweepGarbageCollect.cpp) was dropped in favour of upstream's; the entries above document the design reasoning only. The verification target is now upstream's RunFolderDumpReleaseStep / TickFolderDumpRelease in AssetDumpHandler.cpp and its tests in TestAssetDumpHandler.cpp.
