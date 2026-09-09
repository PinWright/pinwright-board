---
id: B-dump-folder-sweep-never-gcs-ooms-editor
title: "asset.dump_folder never garbage-collects during a sweep — a /Game run (22,446 assets, force) grows the resident set until the editor is OOM-killed, twice in a row"
status: OPEN
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
