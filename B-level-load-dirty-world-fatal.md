---
id: B-level-load-dirty-world-fatal
title: "level.load hard-crashes the editor when a dirty world package is still in memory — Map_Load auto-OKs the 'cannot be unloaded' prompt then fatals on 'World Memory Leaks'"
status: OPEN
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
