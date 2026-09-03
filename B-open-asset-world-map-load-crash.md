---
id: B-open-asset-world-map-load-crash
title: "editor.open_asset on a World hard-crashes the editor — routes maps through OpenEditorForAsset → Map_Load, tripping !LevelList.Contains(TickTaskLevel) during outgoing-world teardown"
status: IN-REVIEW
severity: Critical
category: bug
tags: [editor, open_asset, world, map-load, editor-crash, cold-load, ticktaskmanager, open-asset-world-crash, level-load, level-load-mapswap-crash]
encounters: 5
lastSeen: 2026-08-14T10:42:17+05:00
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
- `#5-ue58-render-thread-ensure` `IN-REVIEW` reporter — Additional UE 5.8 evidence on the same `level.load` cross-map path. `MAP LOAD` from `/Game/DroneFootball/Maps/Football/Bootstrap/L_FootballBootstrap` to `/Game/Maps/New_InfinityMap/L_InfinityMap` began at `05:42:16.665`; immediately after outgoing-world teardown and incoming-world initialization, the render thread raised a handled ensure at `05:42:17.570`: `FAppTime::Get` required `IsInGameThread()` (`Core/Private/Misc/AppTime.cpp:38`) because `UpdateReflectionSceneData` -> `FScene::Update` -> `FScene::Release` executed without an inherited game-thread time context. The ensure stalled reporting until `05:42:23.415` but did not terminate the editor; a later clean restart did not reproduce it. Exact pre-restart evidence: `X:/src/unreal/unreal-fpv-dev/Saved/Logs/DroneFootball-backup-2026.08.14-06.25.43.log:4617-4668`. This does not replace the fatal TickTaskManager repros in #3/#4, but broadens the observed teardown/thread-context damage after synchronous map loading. No source fix or replay attempted in this arena task.
- `#6-safe-point-map-swap` `IN-REVIEW` developer — ROOT CAUSE CONFIRMED AND NARROWED, then fixed at the single shared call site. The trigger is not concurrency between clients and not the saved map's bytes: it is WHEN IN THE FRAME the load runs. `FTickTaskManager::LevelList` holds the frame's `FTickTaskLevel*` values only between `StartFrame` and `EndFrame` — filled by `FillLevelList` (UE 5.8 `TickTaskManager.cpp:2031` -> `:2233 check(!LevelList.Num())` -> `:2240 LevelList.Add`) and emptied at `:2191 LevelList.Reset()` — so `FreeTickTaskLevel`'s `check(!LevelList.Contains(TickTaskLevel))` (`:1992`, called from the C++ destructor `ULevel::~ULevel` at `Level.cpp:470-473`, the only free site in the engine) can ONLY fire while a tick frame is live on the stack. `FEditorFileUtils::LoadMap` -> `Map_Load` (`EditorServer.cpp:2343`) -> `EditorDestroyWorld` (`:2480`) -> `Cleanse` (`EditorEngine.cpp:2763`) -> `CollectGarbage` (`:2859`) destroys the outgoing level synchronously, and PinWright reaches that position by accident: `FRpcDispatcher::ProcessRequest` marshals off-thread requests with `AsyncTask(ENamedThreads::GameThread, ...)` (`RpcDispatcher.cpp:377-385`), and the game thread drains that queue from inside `UWorld::Tick` while waiting on tick groups (`TickTaskManager.cpp:1040 ProcessUntilTasksComplete`, `:1045 WaitUntilTasksComplete`, `:1064 ProcessThreadUntilIdle`, the last reached from the non-blocking `RunTickGroup(TG_DuringPhysics, false)` at `LevelTick.cpp:1765`). So a single client with no concurrency hits it whenever its request happens to land mid-frame — which is why #3/#4 reproduced from a clean editor and why it looks intermittent. The engine documents this exact bug class in its own workaround at `LevelTick.cpp:1891`: "Tick LevelInstanceSubsystem outside of FTickTaskManagerInterface::StartFrame/EndFrame because it can cause levels to be deleted and invalidate its LevelList". Double-add is ruled out (`UWorld::AddLevel` uses `AddUnique`, `FLevelCollection::AddLevel` asserts single ownership, `FillLevelList` asserts the list starts empty); an interrupted incremental visibility update produces a different assertion (`World.cpp:5417-5436`, `:6314`); World Partition changes no mechanism, only the hit rate. FIX: new header `Source/PinWright/Private/Handlers/Level/MapSwapSafePoint.h` carrying the full engine evidence plus `IsAnyWorldTicking()` (walks `GEngine->GetWorldContexts()` for `UWorld::bInTick`, `World.h:1262`), `IsSafeNow()`, and `DeferToSafePoint()` (one-shot zero-interval `FTSTicker`, which `FEngineLoop::Tick` pumps at `LaunchEngineLoop.cpp:6103` — AFTER `GEngine->Tick` at `:5859`, so `LevelList` is provably empty). `level.load` (`LevelHandler.cpp`) now gates its `FEditorFileUtils::LoadMap` on `IsSafeNow()`: inline and responding through `Ctx` exactly as before when safe (so the capture path — test fixtures, text formatters, demo note — and every existing test are unchanged), otherwise deferred and responding through `Ctx.MakeAsyncToken()`. The deferred branch re-enters `FScopedUnattendedRpc`, per the documented continuation gap in `Dispatch/ScopedUnattendedRpc.h`. Exactly one response either way, emitted after LoadMap returns, so the "Synchronous" contract holds and `B-level-load-no-completion-signal` is NOT reopened — completion is never tied to `OnMapOpened`. Responses now carry `deferredToSafePoint`. Because `editor.open_level` and `editor.open_asset` (World) both cross-dispatch into `level.load`, this one call site closes all three verbs — which is the part #3 correctly said the #2 World-routing fix left open. `bInTick` is a conservative gate (set true at `LevelTick.cpp:1564`, before StartFrame; false at `:1965`, after EndFrame); its one hole, `bInTick=false; EnsureCollisionTreeIsBuilt(); bInTick=true;` at `:1752-1754`, is provably unreachable because that function returns immediately in an editor world (`World.cpp:3141-3148`, `if (GIsEditor && !IsPlayInEditor()) return;`) and `LoadMap` refuses to run at all while a PIE world exists (`FileHelpers.cpp:3280`). Tests: `Source/PinWright/Private/Tests/World/TestLevelLoadSafePoint.cpp` — two primitive tests plus `PinWright.level.load.DefersMapSwapToSafePoint`, which drives the REAL registered `level.load` through a genuine `GEditor->NewMap(false)` -> load-back swap with the safe point forced unsafe and asserts the handler returned WITHOUT responding (the counterfactual: revert the branch to a bare `LoadMap` and that assertion fails immediately), then pumps one core-ticker pass and asserts `deferredToSafePoint:true` and `loaded:true`. NOT gate-verified live: the editor was owned by another agent for the duration, so the plugin was not built and no runtime repro was replayed. Needs a tester pass — cross-map `level.load` on a large map (e.g. `/Game/Maps/Dota2_Blockout`) under load, `editor.open_level` both directions, and `editor.open_asset` on a World.
- `#7-runtime-verified-ue58-eacontentexamples58` `IN-REVIEW` tester — GATE-VERIFIED LIVE. The tester pass #6 asked for is done, on UE 5.8 / EAContentExamples58, with the fix built and the full suite green first. Build: clean full-module rebuild with `-DisableAdaptiveUnity` (43 unity blobs, zero standalone `.cpp` actions, so `MapSwapSafePoint.h` is ODR-clean under unity merging), `Result: Succeeded`, `UnrealEditor-PinWright.dll` relinked 2026-08-14 17:52:50. Suite: **3656/3658**, the only failures the pre-existing `PinWright.localization.Validation.{ConfigPath,InvalidRequest}`; all six `PinWright.level.load.*` pass including the three new ones. Runtime: editor launched from the shell as `UnrealEditor.exe <uproject> /Game/Maps/ExampleProjectWelcome -AutoDeclinePackageRecovery` (PID 32580, 18:46:33, ready 18:46:58), then **9 cross-map swaps** alternating `/Game/Maps/ExampleProjectWelcome` <-> `/Game/Maps/Dota2_Blockout` (2122 actors) across all three verbs — `level.load` x5, `editor.open_level` x2, `editor.open_asset` on a World x1, plus the boot load. Every one succeeded and the editor survived all of them. **The deferred branch is confirmed live, not just in the forced-unsafe unit test: 3 of the 9 returned `deferredToSafePoint: true`**, matched one-for-one by 3 log lines `LogPinWrightMapSwap: Map swap requested inside UWorld::Tick; deferring to the next core-ticker pass (destroying the outgoing world's level mid-frame trips !LevelList.Contains(TickTaskLevel)).` So requests really do land mid-frame in ordinary use, and the gate really does catch them. `grep -c "Assertion failed"` over the whole editor log -> **0**; the `!LevelList.Contains(TickTaskLevel)` assertion never fired. `Saved/Crashes/` went 197 -> 199 and BOTH new entries are `CrashType: Ensure` (non-fatal, editor kept running) with **no PinWright frame in either stack**: `..._0000` at 18:46:36 is the startup `r.Mobile.VirtualTextures is deprecated` ensure from `ApplyCVarSettingsFromIni` in `FEngineLoop::PreInitPreStartupScreen`, i.e. before any map work; `..._0001` at 18:47:18 is `FAppTime::Get` requiring `IsInGameThread()` (`AppTime.cpp:38`) on the RENDER thread via `FScene::Release` -> `FScene::Update` -> `UpdateReflectionSceneData` — **the same handled ensure already filed as `#5-ue58-render-thread-ensure`**, now reproduced on a second host and second engine project, and it did not recur across the last 4 swaps (count stayed 199), so it is one-off per session rather than per-swap. Shipped as commit `2586498b` on `master` (pushed, `9da255f6..572c8b65`). Recommend `#5` be split into its own ticket if it is to be pursued: it is an engine scene-release thread-context defect, independent of the map-swap timing this ticket fixed, and it survives the fix.
