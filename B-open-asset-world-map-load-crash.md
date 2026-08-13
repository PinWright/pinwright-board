---
id: B-open-asset-world-map-load-crash
title: "editor.open_asset on a World hard-crashes the editor — routes maps through OpenEditorForAsset → Map_Load, tripping !LevelList.Contains(TickTaskLevel) during outgoing-world teardown"
status: IN-REVIEW
severity: Critical
category: bug
tags: [editor, open_asset, world, map-load, editor-crash, cold-load, ticktaskmanager, open-asset-world-crash, level-load, level-load-mapswap-crash]
encounters: 4
lastSeen: 2026-08-13T22:12:05+05:00
---

# `editor.open_asset` on a World hard-crashes the editor (TickTaskManager assertion during Map_Load teardown)

## Symptom

Opening a World/level asset through `editor.open_asset` fatally crashes the
editor. A cold-booted editor (PinWright subsystem up, empty-args MCP index
returned fine) was pointed at the saved map `/Game/Maps/ExampleProjectWelcome`
via `editor.open_asset` and died with an engine assertion:

```
Assertion failed: !LevelList.Contains(TickTaskLevel)
[File:D:\build\++UE5\Sync\Engine\Source\Runtime\Engine\Private\TickTaskManager.cpp] [Line: 1987]
=== Critical error: ===

FDebug::CheckVerifyFailedImpl2
FTickTaskManager::FreeTickTaskLevel          (TickTaskManager.cpp:1987)
ULevel::~ULevel                              (Level.cpp:466)
FObjectPurge::DestroyObjects / IncrementalPurgeGarbage
CollectGarbage
UEditorEngine::Cleanse
UEditorEngine::EditorDestroyWorld
UEditorEngine::Map_Load                      (EditorServer.cpp:2464)
UAssetDefinition_World::OpenAssets
UAssetEditorSubsystem::OpenEditorForAsset
UnrealEditor-PinWright.dll!AutoHandler_318_  (EditorCommandHandler.cpp:408)
```

The `UnrealEditor.exe` process then exited (post-open check: NO_PROCESS) and the
MCP transport dropped mid-call: "Editor not reachable at
http://127.0.0.1:24966/mcp ... ([WinError 10054] remote host forcibly closed the
connection)". Asset class is `/Script/Engine.World`.

## Root cause

`editor.open_asset` (`Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp:373-420`)
routes **every** asset type — including Worlds — through the generic
`OpenEditorForAsset` path with no special-casing for `UWorld`:

```cpp
  UObject *Asset = UEditorAssetLibrary::LoadAsset(AssetPath);   // line 402
  if (!Asset) {
    Ctx.SendError(TEXT("LOAD_FAILED"), TEXT("Failed to load asset"));
    return true;
  }

  const bool bOpened = AssetEditorSS->OpenEditorForAsset(Asset); // line 408 — crash site
```

For a World, `UAssetEditorSubsystem::OpenEditorForAsset` dispatches to
`UAssetDefinition_World::OpenAssets`, which performs a full
`UEditorEngine::Map_Load` (the same thing double-clicking a `.umap` does). That
`Map_Load` first tears down and GCs the currently-loaded world
(`EditorDestroyWorld` → `Cleanse` → `CollectGarbage` → `~ULevel` →
`FreeTickTaskLevel`), and the outgoing world's level is still registered in the
tick-task manager's `LevelList`, tripping the `!LevelList.Contains(TickTaskLevel)`
assertion.

This host makes the crash acute: `/Game/Maps/ExampleProjectWelcome` is the
editor **startup map** (`Config/DefaultEngine.ini`:
`EditorStartupMap=/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome`), so a
cold-booted editor already has that exact world live. `editor.open_asset` on it
forces `Map_Load` to reopen the already-active map, tearing down the live world
and hitting the assertion during that GC.

## What should happen

`editor.open_asset` should not be able to hard-crash the editor. Two directions
for the fixer to weigh (the guilty line, EditorCommandHandler.cpp:408, is common
to both):

1. **open_asset should special-case Worlds.** A `UWorld` target should route
   through the map-load-with-proper-teardown path (the same resolution
   `editor.open_level` uses — its dedicated sibling verb for maps) or be rejected
   with a clean error steering the caller to `editor.open_level`, rather than
   handed to the generic `OpenEditorForAsset` that Map_Loads over the live world.
   Re-opening the *already-active* map in particular should be a no-op, not a
   destroy-and-reload.
2. **Save-time integrity angle.** This surfaced through the workflow's
   cold-restart CorruptionCheck on a map the attempt had just re-lit and saved
   (`level.save` → `saved:true`). The save path has no World-level integrity gate
   analogous to the Blueprint one, so if the lighting mutations left the world's
   level tick-registration inconsistent, nothing at save time would catch it. The
   fixer should determine whether the tick-task inconsistency is inherent to
   `Map_Load`-over-the-active-map (hypothesis 1, expected to reproduce on a
   pristine map too) or specific to the saved world state (hypothesis 2).

The crash callstack points primarily at hypothesis 1: the assertion fires in the
teardown of the **outgoing** world during `Map_Load`, before the saved map's data
is deserialized into the editor world — so the trigger is the open/Map_Load path,
not (yet) the saved map's on-disk contents.

## Repro (confirmed by a real cold editor restart)

1. Attempt re-lit and saved `/Game/Maps/ExampleProjectWelcome`
   (`lighting.spawn_light`, `lighting.ensure_single_sky_light`,
   `lighting.configure_shadows`, `lighting.setup_global_illumination`,
   `lighting.setup_volumetric_fog`, `lighting.set_ambient_occlusion`,
   `lighting.set_exposure`, then `level.save` → job `completed`, `saved:true`).
2. Editor quit + relaunched headless WITHOUT baseline restore. Cold boot was
   clean: PinWrightSubsystem initialized, empty-args MCP index returned.
3. `editor.open_asset {assetPath:"/Game/Maps/ExampleProjectWelcome"}` →
   editor crashed with the assertion above; process exited; MCP transport
   dropped ([WinError 10054]).

(The restart is itself the replay-confirmation; not re-replayed through the MCP
to avoid re-crashing the editor.)

severity rationale: impact=corruption/editor-crash × reach=every-session (opening
a level asset is a routine action; here it is also the startup map) -> Critical

## History
- `#3-additional-level-load-direct-mapswap-crash` `IN-REVIEW` reporter — Additional evidence (NEW ANGLE — broadens the affected-method set AND bears directly on the #2 fix). The SAME fatal assertion `!LevelList.Contains(TickTaskLevel)` (TickTaskManager.cpp:1987) also fires when `level.load` calls `FEditorFileUtils::LoadMap` DIRECTLY — not only via `editor.open_asset` -> `OpenEditorForAsset` -> `Map_Load`. A realism attempt applied KillZ/gravity WorldSettings edits to `/Game/Maps/ExampleProjectWelcome` and persisted them (editor.save_all survived a disk reload); then, while re-testing `level.save` with distinctive values, it issued a routine `level.load` map-swap and the editor hard-crashed. The MCP went unreachable (connection refused on repeated probes); the ~20s+10s backoff canary (`mcp__pinwright__call` no-args) did NOT recover -> editor genuinely died. Fresh evidence THIS iteration: crash dump `Saved/Crashes/UECC-Windows-A6B5BD09422B63D2B71C1DBFDC523483_0000/` (CrashContext.runtime-xml + UEMinidump.dmp + crash .log, mtime 22:32) and the live editor log `Saved/Logs/EAContentExamples57.log` (assertion at 19.32.28, Critical error block at 19.32.32). GUILTY SOURCE LINE (read from plugin source, not inferred): `Plugins/PinWright/Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:241` -> `FEditorFileUtils::LoadMap(FileToLoad);` (the `level.load` handler; the callstack return address resolves to LevelHandler.cpp:243, the line right after the call). Verbatim assertion + callstack from the editor log:

```
appError called: Assertion failed: !LevelList.Contains(TickTaskLevel) [File:D:\build\++UE5\Sync\Engine\Source\Runtime\Engine\Private\TickTaskManager.cpp] [Line: 1987]
=== Critical error: ===
Assertion failed: !LevelList.Contains(TickTaskLevel) [File:D:\build\++UE5\Sync\Engine\Source\Runtime\Engine\Private\TickTaskManager.cpp] [Line: 1987]

FDebug::CheckVerifyFailedImpl2()                                          AssertionMacros.cpp:745
FTickTaskManager::FreeTickTaskLevel()                                     TickTaskManager.cpp:1987
ULevel::~ULevel()                                                         Level.cpp:466
ULevel::`scalar deleting destructor'()
FObjectPurge::DestroyObjects()                                           GarbageCollection.cpp:914
IncrementalPurgeGarbage()                                                GarbageCollection.cpp:4721
CollectGarbage()                                                         GarbageCollection.cpp:6216
UEditorEngine::Cleanse()                                                 EditorEngine.cpp:2859
UEditorEngine::EditorDestroyWorld()                                      EditorServer.cpp:2064
UEditorEngine::Map_Load()                                               EditorServer.cpp:2464
UEditorEngine::HandleMapCommand() / Exec_Editor()                        EditorServer.cpp:6227 / 5688
FEditorFileUtils::LoadMap()                                             FileHelpers.cpp:3306
UnrealEditor-PinWright.dll!AutoHandler_400_()  [level.load]             LevelHandler.cpp:243
FRpcDispatcher::DrainAutoRegistrations lambda                           RpcDispatcher.cpp:298
FRpcDispatcher::ProcessRequest()                                        RpcDispatcher.cpp:480
TGraphTask<FAsyncGraphTask>::ExecuteTask() / FNamedTaskThread::ProcessTasks...   (task graph)
FTickTaskSequencer::ReleaseTickGroup()                                  TickTaskManager.cpp:1035
FTickTaskManager::RunTickGroup()                                        TickTaskManager.cpp:2129
UWorld::Tick()                                                          LevelTick.cpp:1848
UEditorEngine::Tick() / UUnrealEdEngine::Tick()                        EditorEngine.cpp:1961
```

  Note the LOWER frames: the `level.load` RPC was processed REENTRANTLY from inside a tick group (`FTickTaskManager::RunTickGroup` -> `FTickTaskSequencer::ReleaseTickGroup` -> task graph -> `FRpcDispatcher::ProcessRequest`), so `Map_Load` tore down the outgoing world while its level was still registered in the tick-task manager's `LevelList` -> assertion. This is the same engine root cause as #1/#2 (Map_Load destroying a still-tick-registered live world's level), reached through a different verb and trigger. IMPACT ON THE #2 FIX: #2 routes `editor.open_asset` Worlds -> `editor.open_level` -> `level.load` on the premise that `level.load` is the "vetted safe" path (no-ops via `DoesRequestedLevelMatchCurrentWorld` when the map is already current). That no-op only covers the already-current case; on a GENUINE map-swap `level.load` performs the same destroy-and-Map_Load and trips the SAME assertion — so redirecting the crash INTO `level.load` does not by itself make Worlds safe. The fixer must guard the Map_Load teardown for BOTH verbs (e.g. never destroy/GC the outgoing world while a tick group is in flight — defer `level.load`'s `FEditorFileUtils::LoadMap` to a clean point outside `RunTickGroup`). Affected methods now: `editor.open_asset` (World), `level.load` (direct), and by cross-dispatch `editor.open_level`. encounters 1 -> 2; severity unchanged. severity rationale: impact=crash x reach=every-session (level.load / map-swap is a routine action) -> Critical.
- `#2-fix-route-world-to-open-level` `IN-REVIEW` developer — GO (hypothesis 1; severity Critical unchanged, full scope, no splits). `editor.open_asset` now special-cases `UWorld`: after `LoadAsset` it detects an `Asset->IsA(UWorld::StaticClass())` target and cross-dispatches to the dedicated `editor.open_level` verb (`Ctx.GetSubsystem()->DispatchMethod("editor.open_level", ...)` with the World's package path) instead of handing it to `OpenEditorForAsset -> Map_Load`. `editor.open_level` delegates to `level.load`, which no-ops via `DoesRequestedLevelMatchCurrentWorld` when the requested map is already active — so reopening the live startup map is now a no-op, never a destroy-and-reload. Fixes only the crashing self-reload case; the working different-map load is preserved through the vetted level.load path. Hypothesis 2 (World save-time integrity gate) deliberately NOT pursued (ruled out as the trigger — the assertion fires in outgoing-world teardown, not from saved bytes). File: `Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp` (~408 guard). Test: `PinWright.editor.open_asset.WorldDoesNotCrashEditor` (`Tests/EditorOps/TestOpenAssetWorldNoCrash.cpp`, adopted+strengthened to drive the handler through the live subsystem so the real cross-dispatch runs). Verified: plugin compiles clean; differential observed via stash — pre-fix `Result={Fail}` (live world torn down + swapped), post-fix `Result={Success}`.
- `#1-initial-repro` `OPEN` reporter — Cold-restart CorruptionCheck crashed the freshly cold-booted editor when `editor.open_asset` opened the saved startup World `/Game/Maps/ExampleProjectWelcome`. Fatal assertion `!LevelList.Contains(TickTaskLevel)` (TickTaskManager.cpp:1987) fires in `FTickTaskManager::FreeTickTaskLevel` during `~ULevel` GC inside `UEditorEngine::Map_Load` teardown, dispatched from `editor.open_asset` (EditorCommandHandler.cpp:408, `AutoHandler_318_`). Confirmed via source read: `editor.open_asset` routes ALL asset types through `OpenEditorForAsset`, which for a World does a full `Map_Load` that tears down the live world; the target here is also the `EditorStartupMap`, so the cold editor already had that world loaded and open_asset forced a destroy-and-reload. Leading hypothesis is that `editor.open_asset` must special-case Worlds (route to the map-load path like `editor.open_level`, no-op on the active map, or reject) rather than Map_Load over the live world; secondary hypothesis is a missing World save-time integrity gate. Distinct from `B-open-level-engine-mount-mangled` (that is `editor.open_level` path-string mangling, no crash) and from the Widget-Blueprint cold-load corruption in `B-bp-saved-state-corruption-mcp-edits` (different asset type + crash signature).
- `#4-ue58-open-level-repro` `IN-REVIEW` reporter — Reproduced the same fatal cross-map teardown twice through the public `editor.open_level` RPC on the UE 5.8 DroneFootball host, including once from a clean freshly restarted editor. First, after deleting the only transient review actor from `/Game/Maps/New_InfinityMap/L_InfinityMap`, `editor.open_level {levelPath:"/Game/DroneFootball/Maps/Football/Bootstrap/L_FootballBootstrap"}` dropped the MCP stream with WinError 10054 and exited only the Football editor. After restart, `editor.list_dirty_packages` returned `count:0`, PIE was false, and the tagged actor count was zero; the reverse `editor.open_level {levelPath:"/Game/Maps/New_InfinityMap/L_InfinityMap"}` crashed identically, ruling out unsaved content as the trigger. Exact log: `X:/src/unreal/unreal-fpv-dev/Saved/Logs/DroneFootball.log`, 2026-08-13 17:11:57-17:12:05 UTC. Assertion is `!LevelList.Contains(TickTaskLevel)` at UE 5.8 `TickTaskManager.cpp:1992`; the callstack is `FTickTaskManager::RunTickGroup` -> `FRpcDispatcher::ProcessRequest` -> `EditorCommandHandler.cpp:600` cross-dispatch -> `LevelHandler.cpp:249` `FEditorFileUtils::LoadMap` -> `Map_Load` -> `EditorDestroyWorld` -> `CollectGarbage` -> `ULevel::~ULevel` -> `FreeTickTaskLevel`. Crash reports were written under `Saved/Crashes/` at local times 22:10:38 and 22:12:03. This directly confirms the #3 reentrancy diagnosis and that the current IN-REVIEW World-routing fix is incomplete until actual map swaps are deferred outside the active tick group. No fix attempted in this capture-validation task.
