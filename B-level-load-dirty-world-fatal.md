---
id: B-level-load-dirty-world-fatal
title: "level.load hard-crashes the editor when a dirty world package is still in memory — Map_Load auto-OKs the 'cannot be unloaded' prompt then fatals on 'World Memory Leaks'"
status: IN-REVIEW
severity: Critical
category: bug
tags: [level, level-load, open-level, editor-crash, map-load, dirty-package, unattended, shared-editor]
encounters: 2
lastSeen: 2026-09-02T19:12:00Z
---

# `level.load` fatals with `World Memory Leaks` when an in-memory world package is dirty

## Symptom

`level.load` on a shared editor killed the whole editor process (and with it every
agent working in it). Distinct crash signature from the already-tracked
`B-open-asset-world-map-load-crash` family: that one is the tick-reentrancy
assertion `!LevelList.Contains(TickTaskLevel)` in `FreeTickTaskLevel`. **This one
is a different fatal, and it fired on the deferred safe-point path — i.e. the
`#6-safe-point-map-swap` fix worked and the editor still died.**

Verbatim, from `Saved/Logs/EAContentExamples58.log:4801-4855` (UTC):

```
[19.09.03:546] LogPinWrightSafePoint: Tick-unsafe work requested inside UWorld::Tick; deferring to
  the next core-ticker pass (level.load: FEditorFileUtils::LoadMap tears the outgoing world down
  and GCs its ULevel).
[19.09.03:557] Cmd: MAP LOAD FILE=".../Content/FPS/Maps/FPS_Compound.umap" TEMPLATE=0 SHOWPROGRESS=1
[19.09.03:624] Message dialog closed, result: Ok, title: Message, text: The following assets have
  been modified and cannot be unloaded:
    /Game/FPS/Maps/FPS_Compound
  Saving these assets will allow them to be unloaded.
[19.09.03:673] LogEditorServer: Error: Old world /Game/FPS/Maps/FPS_Compound.FPS_Compound not
  cleaned up by garbage collection while loading new map!
[19.09.03:702] LogEditorServer: Error: Old level package /Game/FPS/Maps/FPS_Compound not cleaned up
  by garbage collection while loading new map!
[19.09.03:702] LogWindows: Error: appError called: Fatal error:
  [File:...\Editor\UnrealEd\Private\EditorServer.cpp] [Line: 2544]
  World Memory Leaks: 2 leaks objects and packages. See The output above.
```

Callstack (same log, 4842-4855), PinWright frames verbatim:

```
UEditorEngine::Map_Load()                    EditorServer.cpp:2548
UEditorEngine::HandleMapCommand()            EditorServer.cpp:6445
UEditorEngine::Exec_Editor()                 EditorServer.cpp:5906
FEditorFileUtils::LoadMap()                  FileHelpers.cpp:3304
UnrealEditor-PinWright.dll!AutoHandler_314_  Handlers/Level/LevelHandler.cpp:269
UnrealEditor-PinWright.dll!AutoHandler_314_  Handlers/Level/LevelHandler.cpp:294
PinWrightSafePoint::RunAtSafePoint           Dispatch/SafePoint.h:470
PinWrightSafePoint::DeferToSafePoint         Dispatch/SafePoint.h:326
FTSTicker::Tick()                            Ticker.cpp:121
FEngineLoop::Tick()                          LaunchEngineLoop.cpp:6104
```

The process then ran `StaticShutdownAfterError` and requested exit. The PinWright
gateway on port 27145 went to connection-refused; `Get-Process UnrealEditor` still
listed PID 24560 (sitting in post-fatal shutdown), so a liveness check by process
name reports "running" while every RPC fails.

## Root cause

`Handlers/Level/LevelHandler.cpp:269`:

```cpp
    auto PerformSwapAndBuildResponse = [LevelPath, FileToLoad](bool bDeferred)
    {
        FlushRenderingCommands();
        FEditorFileUtils::LoadMap(FileToLoad);      // <-- line 269, no dirty-world precondition
```

The handler gates the call on **tick safety** (`PinWrightSafePoint::RunAtSafePoint`,
added by this ticket family's `#6`) but has **no precondition on whether the
outgoing / still-resident world packages can actually be purged**. When a world
package is dirty and still referenced, `Map_Load` prompts
("The following assets have been modified and cannot be unloaded"), that prompt is
auto-answered `Ok` under unattended operation, `EditorDestroyWorld` +
`CollectGarbage` then cannot free the old world, and `Map_Load` reaches its
`World Memory Leaks` fatal at `EditorServer.cpp:2544` — an unconditional
`UE_LOG(Fatal)`, not a recoverable error. Under unattended automation the prompt
that would let a human hit Cancel is exactly the safety valve that gets removed, so
the fatal is the *only* reachable outcome.

The plugin already owns every piece needed to refuse this cleanly before calling
`LoadMap`: `editor.list_dirty_packages` enumerates dirty packages, and
`EditorLoadingAndSavingUtils::GetDirtyMapPackages()` names the dirty *world*
packages specifically.

## What should happen

`level.load` (and by cross-dispatch `editor.open_level` and `editor.open_asset` on a
World) must never be able to hard-kill the editor. Before `FEditorFileUtils::LoadMap`,
check for dirty map packages that the incoming load will have to purge, and:

- return a typed error (e.g. `DIRTY_WORLD_BLOCKS_MAP_SWAP`) naming each dirty world
  package and pointing at `level.save` / `editor.save_all`; or
- accept an explicit opt-in argument (`saveDirtyWorlds: true` → save them first,
  `discardDirtyWorlds: true` → mark clean and proceed) so the caller chooses.

A refusal is strictly better than a fatal: the caller loses one call, not the editor
and every other agent in it. This matters most in exactly the configuration PinWright
is used in here — one shared editor, several agents — where a single `level.load` from
one stream destroys six other streams' unsaved in-memory work.

## Repro

1. Load map A; make an actor/world edit that dirties A's package; do not save.
2. `level.load` map B (or the same A after another world became active in between,
   which is what happened here — the editor world was `/Game/FPS/Test/T_AI` and the
   dirty resident package was `/Game/FPS/Maps/FPS_Compound`).
3. Editor fatals at `EditorServer.cpp:2544` `World Memory Leaks`.

**Workaround:** call `editor.list_dirty_packages` (or
`EditorLoadingAndSavingUtils.get_dirty_map_packages()` via `python.execute`) before
every `level.load`, and `editor.save_all` / explicitly clean any dirty world package
first. There is no way to recover after the call is made.

severity rationale: impact=editor crash (and loss of every unsaved in-memory edit in
a shared editor) x reach=every-session (a map swap is routine, and dirty worlds are
the normal state mid-authoring) -> Critical.

## History
- `#1-filed` `OPEN` reporter — Hit while starting the UI stream on EAContentExamples58 (UE 5.8, shared editor, port 27145). My own RPCs (`editor.pie_status`, `asset.exists`, `property.get`) succeeded at ~19:08 UTC; at 19:09:03 another stream's `level.load` of `/Game/FPS/Maps/FPS_Compound` took the deferred safe-point branch, `Map_Load` auto-OK'd the "assets have been modified and cannot be unloaded" dialog for the still-resident dirty `/Game/FPS/Maps/FPS_Compound` package, GC could not free the old world or its package, and `UEditorEngine::Map_Load` hit the unconditional `World Memory Leaks: 2 leaks objects and packages` fatal at `EditorServer.cpp:2544`. Evidence: `X:/src/unreal/EAContentExamples58/Saved/Logs/EAContentExamples58.log:4801-4855` (dialog at :4808, fatal at :4835, PinWright frames at :4849-4852). Guilty line read from source, not inferred: `Plugins/PinWright/Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:269` `FEditorFileUtils::LoadMap(FileToLoad);` — the handler gates tick-safety but has no precondition on purgeable/dirty world packages. Explicitly NOT a duplicate of `B-open-asset-world-map-load-crash`: that ticket's signature is the `!LevelList.Contains(TickTaskLevel)` assertion from tick reentrancy, and its `#6` safe-point fix was ACTIVE here (the log shows the deferral line firing) — this is a second, independent fatal on the same call. After the crash the gateway refused connections while `Get-Process UnrealEditor` still listed the PID, so process-name liveness checks report a false "running".
- `#2-audio-stream-also-killed` `OPEN` reporter — Same fatal, independently observed from the AUDIO stream of the same six-agent editor, which confirms the blast radius is every attached agent rather than the caller alone. My own `audio.synth.generate` / `audio.synth.describe_schema` calls had been succeeding continuously until 19:09; the next call after the crash returned `EDITOR_NOT_RUNNING: ... (connection refused)`. Log tail matches `#1` line for line: the "assets have been modified and cannot be unloaded: /Game/FPS/Maps/FPS_Compound" dialog auto-answered `Ok`, `LogReferenceChain` showing the world unreachable but holding `GARBAGE_COLLECTION_KEEPFLAGS`, both `LogEditorServer: Error: Old world / Old level package ... not cleaned up by garbage collection while loading new map!` lines, then `appError` at `EditorServer.cpp:2544` with `LevelHandler.cpp:269/:294` in the PinWright frames. One detail worth adding to the false-liveness note in `#1`: `Get-Process UnrealEditor` did not merely still list the PID, it reported `Responding: True` for it, so even a hung-window check passes on a process that is inside `StaticShutdownAfterError`. The only reliable probe is the port itself (`Test-NetConnection -Port 27145`, which returned False) or a real RPC. Cost here: the whole stream stalled with no way to self-recover, since agents in this workflow are explicitly forbidden from calling `editor_start`/`editor_restart`, so a single agent's `level.load` converts into a hard stop for everyone until a human or the orchestrator intervenes. Reinforces the ask in `#1`: refuse the load with a typed error when any world package is dirty, rather than letting `Map_Load` reach an unconditional `appError`.
- `#3-corrects-false-liveness-detail-in-2` `OPEN` reporter — **Correcting my own `#2`.** The claim there that `Get-Process UnrealEditor` reported `Responding: True` *for the crashed editor* is WRONG and should not be relied on: the orchestrator confirmed the PID I read (24560) belongs to a different project entirely — an `EAContentExamples57` UE 5.7 automation run on the same machine — while the actual crashed editor for this project (PID 97980) had already exited and was absent from the process list. Everything else in `#2` stands (the fatal, the log chain, the blast radius across every attached agent, the stall with no self-recovery). The trap is real but it is a *different* trap than I described, and arguably a worse one: `Get-Process UnrealEditor` is matched **by process name**, so on a machine running more than one UE project it happily returns a healthy stranger's editor and reports it `Responding: True`. An agent that liveness-checks by process name therefore concludes the editor is up when its own editor is gone, and — worse for a kill-and-restart flow — a name-matched kill would take out an unrelated project's session. The only sound probes remain the gateway port (`Test-NetConnection -Port 27145`, which correctly returned False throughout) or a real RPC; if a PID check is wanted it must be matched on the project path in the command line, never on the image name. Recording this here so the next reader of `#2` does not inherit the wrong mental model.

## Fix

TRUE as filed, and verified against engine source rather than inferred. The reachability is
narrower than the ticket states, and the fix is scoped to what is actually fatal.

**Root cause.** `UEditorEngine::Map_Load` checks only the **TARGET** package, not "any dirty
world". UE 5.8 `EditorServer.cpp`: `:2399` `ExistingPackage = FindPackage(nullptr, *LongTempFname)`
-> `:2494` `UPackageTools::UnloadPackages(WorldPackages)` -> `:2498` re-find -> `:2520`
`if ((ExistingWorld && !IsWorldValidForReuse(ExistingWorld)) || (ExistingPackage && !ExistingWorld))`
-> `:2544` unconditional `UE_LOGF(Fatal)`. `UPackageTools::UnloadPackages` skips dirty packages by
POLICY (`PackageTools.cpp:381 if (!Params.bUnloadDirtyPackages && TopLevelPackage->IsDirty())`), so
a resident-and-dirty target survives the unload and reaches the appError. The OUTGOING world is not
implicated: `EditorDestroyWorld` strips it itself (`:1990-1991 ClearFlags(RF_Standalone |
RF_Transactional)` + `RemoveFromRoot()`, `:2009 SetFlags(RF_Transient)`), which is why `T_AI`
cleaned up fine in the shipped log while `FPS_Compound` did not.

Consequence for the design: a guard on "any dirty world package" (as `#1` proposed) would refuse
the ordinary safe swap - edit map A, open map B - and make the verb unusable. The guard blocks
exactly `resident && dirty && !isCurrentEditorWorld && (!worldFound || worldEverInitialized)`. The
`!worldEverInitialized` exemption is load-bearing: `:2470 IsWorldValidForReuse` keeps and reuses an
uninitialized world, so `level.create` -> `level.load` must not be refused.

**Design.** Refusal is the default, not a save: in the shared-editor configuration the dirty map
usually belongs to another agent, and saving it publishes their half-finished work. `level.load`
now emits `DIRTY_WORLD_BLOCKS_MAP_SWAP` with a payload naming `blockingPackage` /
`packageDirty` / `worldFound` / `worldEverInitialized`, having changed nothing. One opt-in,
`saveDirtyTargetWorld:true`, saves that ONE package and then loads (narrower than `editor.save_all`,
and it PRESERVES the caller's unsaved edits rather than discarding them) - the same
refuse-by-default + named-override shape `asset.save` uses for `SAVE_DISK_STATE_DIVERGED` /
`overwriteDiskChanges` and `level.delete` for `ASSET_IN_USE` / `force`. No `discard` mode was added:
it destroys data and has no reported use case. The check runs INSIDE the safe-point body, as the
last instruction before `LoadMap`, so a shared editor cannot dirty the target in the window a
pre-`RunAtSafePoint` check would leave open. `editor.open_level` and `editor.open_asset` (World)
cross-dispatch into the same call site and are covered; the opt-in itself is only on `level.load`.

**Files changed** (plugin repo, uncommitted):
- `Plugins/PinWright/Source/PinWright/Private/Utils/MapSwapDirtyWorldGuard.h` (new) - the
  precondition, with the engine mechanism and every non-blocking case documented file:line.
- `Plugins/PinWright/Source/PinWright/Private/Utils/MapSwapDirtyWorldGuard.cpp` (new) - pure
  predicate, read-only probe (`FindPackage`, never `LoadPackage`; resolves the SAME string handed
  to `LoadMap` through `FPackageName::TryConvertToMountedPath`), and the opt-in save whose verdict
  is a re-probe rather than `PromptForCheckoutAndSave`'s return code.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Level/LevelHandler.cpp` - `level.load`
  summary + `saveDirtyTargetWorld` param + the guard inside the safe-point body.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/ErrorCodes.h` -
  `ERR_DIRTY_WORLD_BLOCKS_MAP_SWAP`.
- `Plugins/PinWright/Docs/wiki-src/level.md` - `### level.load` paragraph.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Core/TestMapSwapDirtyWorldGuard.cpp` (new) -
  3 tests: the decision table (every branch of the leak check), the probe driven against a real
  resident dirty package in both package-path and .umap-filename form, and the negative side
  (absent / unmounted / empty paths never block).

**Verification a reviewer should run.**
1. Compile, then `PinWright.core.map_swap_guard` (3 tests) plus `PinWright.level` and
   `PinWright.core.error_codes` (the registry walk sees the new code).
2. Live check, which the tests deliberately do NOT do (driving it pre-fix crashes the editor):
   `level.create {levelPath:"/Game/Maps/__GuardProbe"}` -> `level.save` -> spawn an actor ->
   `level.load` some other map -> `level.load {levelPath:"/Game/Maps/__GuardProbe"}`. Expect
   `DIRTY_WORLD_BLOCKS_MAP_SWAP` naming `/Game/Maps/__GuardProbe`, editor still alive, and
   `editor.list_dirty_packages` unchanged. Then retry with `saveDirtyTargetWorld:true` and expect
   `loaded:true` + `savedDirtyTargetWorld:true`, with the spawned actor still present after the
   swap.
3. Non-regression: `level.create` followed immediately by `level.load` of the same path must still
   work (uninitialized world -> reused, must NOT be refused).

**Known residual, stated rather than hidden.** A resident target that is CLEAN but pinned by a real
reference still reaches `:2544`. No flag predicts that, the guard does not claim to cover it, and
the suite has already met it once - see the `FNiagaraWorldManager` teardown note at
`Tests/World/TestLevelHandlers.cpp:916`.

## Fix correction (post-suite)

The first cut of the guard was **over-broad and broke two passing tests** —
`PinWright.level.structure.create_level_instance.SetsWorldAsset` and
`...create_packed_level_actor.PacksSourceLevel` in `Saved/Logs/pw_wave_suite.log` at
`07.22.06`. Corrected in the same working tree; recording the mistake because the missing fact
is the interesting part.

**What went wrong.** The predicate read the dirty flag but not whether the resident world would
still exist when the leak check runs. `Map_Load` order is: `EditorDestroyWorld` (`:2480`, which
runs `Cleanse` -> `CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS)`) **then** the
`TObjectIterator<UWorld>` sweep (`:2485`), the unload (`:2494`) and the check (`:2520`). A world
already unrooted and stripped of `RF_Standalone` is reclaimed by that first collect and never
reaches the sweep, so its dirty flag cannot produce the fatal.

That is exactly the state `level.structure.create_level` leaves behind: it swaps the active
world by hand and tears the outgoing one down with `UWorld::DestroyWorld`, which does
`RemoveFromRoot()` + `ClearFlags(RF_Standalone)` (`World.cpp:2792-2793`) but runs **no** collect
(`LevelStructureHandler.cpp` says so in its own comment). The suite's
`FScopedEditorWorldMapGuard` then calls `level.load` to restore the original map; the guard
refused it, the restore silently failed (the harness uses a bare `FHandlerContext`, so the error
was dropped — `HandlerContext::SendError called with no Subsystem and no ResponseCapture` at
`07.22.05:118`, with no `Cmd: MAP LOAD FILE=` after it), the editor was stranded on a torn-down
probe world, and the next two tests failed against it. Every earlier restore in the same log
shows the `SendSuccess` variant plus a real `MAP LOAD`, which is what pins the regression on this
change rather than on the wave's other edits.

**The corrected discriminator is the engine's own,** printed verbatim in the shipped crash log
for the world that DID leak: *"is not currently reachable but it does have some of
GARBAGE_COLLECTION_KEEPFLAGS set"* (= `RF_Standalone` in the editor, `GarbageCollection.h:28`).
`FTargetWorldState` gained `bWorldSurvivesEditorCollect` (`World->IsRooted() ||
World->HasAnyFlags(RF_Standalone)`), and a dirty resident world that fails it is no longer
refused. `FPS_Compound` — a normally-loaded, never-destroyed map world — keeps `RF_Standalone`
and is still refused, so the fix is unchanged for the reported defect.

Second change made at the same time: `WouldMapLoadFatal` now takes `const FTargetWorldState&`
instead of a row of positional bools. Six near-synonymous booleans in a fixed order is a
transposition waiting to happen on a guard whose job is preventing a process kill; the struct
form makes every test case name its fields.

Files touched by the correction: `Utils/MapSwapDirtyWorldGuard.h` / `.cpp`,
`Handlers/Level/LevelHandler.cpp` (new `worldSurvivesGarbageCollect` field in the refusal
payload), `Tests/Core/TestMapSwapDirtyWorldGuard.cpp` (new regression case
`garbage-in-waiting world (unrooted, no RF_Standalone) must NOT be refused`), and the
`Docs/wiki-src/level.md` paragraph.

A reviewer should re-run `PinWright.level` and confirm `create_level_instance.SetsWorldAsset`
and `create_packed_level_actor.PacksSourceLevel` are green, and that a `Cmd: MAP LOAD FILE=`
line follows each `create_level` test's teardown in the log.

**Separately, and NOT fixed here:** the harness's map-restore guard
(`Tests/TestWorldUtils.h`, `FScopedEditorWorldMapGuard::~FScopedEditorWorldMapGuard`) discards
`level.load`'s result entirely, so a refused restore is invisible and shows up several tests
later as an unrelated failure. That cost the whole diagnosis here. Worth its own ticket.
