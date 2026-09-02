---
id: B-level-load-dirty-world-memory-leak-fatal
title: "level.load kills the editor with `World Memory Leaks: 2 leaks objects and packages` when the TARGET map's world is already loaded and dirty — the unattended auto-answered `cannot be unloaded` dialog turns a refusable condition into an appError"
status: OPEN
severity: Critical
category: bug
tags: [level, level-load, editor-crash, map-load, world-memory-leaks, dirty-package, unattended, shared-editor, EditorServer-2544]
---

# `level.load` hard-crashes the editor when the target world is already in memory and dirty

## Symptom

A shared UE 5.8 editor (PinWright gateway on port 27145, six agents attached) died
outright mid-session. The last RPC was a `level.load` of `/Game/FPS/Maps/FPS_Compound`
issued while the editor had `/Game/Maps/ExampleProjectWelcome` booted, `T_AI` as the
live world, **and an already-loaded, already-dirty in-memory `FPS_Compound` world**.

```
Cmd: MAP LOAD FILE=".../Content/FPS/Maps/FPS_Compound.umap" TEMPLATE=0 SHOWPROGRESS=1 FEATURELEVEL=4
LogWorld: UWorld::CleanupWorld for T_AI, bSessionEnded=true, bCleanupResources=true
Message dialog closed, result: Ok, title: Message, text: The following assets have been modified and cannot be unloaded:
    /Game/FPS/Maps/FPS_Compound
Saving these assets will allow them to be unloaded.
LogReferenceChain: (standalone) World /Game/FPS/Maps/FPS_Compound.FPS_Compound is not currently reachable but it does have some of GARBAGE_COLLECTION_KEEPFLAGS set.
LogEditorServer: Error: Old world /Game/FPS/Maps/FPS_Compound.FPS_Compound not cleaned up by garbage collection while loading new map!
LogEditorServer: Error: Old level package /Game/FPS/Maps/FPS_Compound not cleaned up by garbage collection while loading new map! Referenced by:
 (standalone)  World /Game/FPS/Maps/FPS_Compound.FPS_Compound
 -> UObject* UWorld:: =  Package /Game/FPS/Maps/FPS_Compound
LogWindows: Error: appError called: Fatal error:
  [File:.../Editor/UnrealEd/Private/EditorServer.cpp] [Line: 2544]
  World Memory Leaks: 2 leaks objects and packages. See The output above.
```

Callstack (top frames):

```
UEditorEngine::Map_Load()                                   EditorServer.cpp:2548
UEditorEngine::HandleMapCommand()                           EditorServer.cpp:6445
UEditorEngine::Exec_Editor()                                EditorServer.cpp:5906
FEditorFileUtils::LoadMap()                                 FileHelpers.cpp:3304
UnrealEditor-PinWright.dll!AutoHandler_314_ lambda_1        Handlers/Level/LevelHandler.cpp:269
UnrealEditor-PinWright.dll!AutoHandler_314_ lambda_2        Handlers/Level/LevelHandler.cpp:294
PinWrightSafePoint::RunAtSafePoint lambda                   Dispatch/SafePoint.h:470
PinWrightSafePoint::DeferToSafePoint lambda                 Dispatch/SafePoint.h:326
FTSTicker::Tick()                                           Ticker.cpp:121
FEngineLoop::Tick()                                         LaunchEngineLoop.cpp:6104
```

The editor process then executed `StaticShutdownAfterError` and exited. Every agent
attached to that editor lost its session; the next `call()` on any method returned
`EDITOR_NOT_RUNNING ... (connection refused)`.

Exact log: `X:/src/unreal/EAContentExamples58/Saved/Logs/EAContentExamples58.log`,
`2026.09.02-19.09.03` .. `19.09.12` UTC (lines around the last `MAP LOAD FILE=` entry).

## What I expected

`level.load` refuses, or saves/discards first, when the target map's world is already
resident and dirty. What is not acceptable is issuing `MAP LOAD` into a condition the
engine treats as fatal, in a process shared by other agents.

## Distinct from `B-open-asset-world-map-load-crash`

That ticket's signature is the `!LevelList.Contains(TickTaskLevel)` assertion
(`TickTaskManager.cpp:1992`) caused by a **mid-frame** map swap, and its `#6`/`#7`
fix defers `LoadMap` to a core-ticker safe point. That fix is present and working
here — the callstack shows the deferred path (`SafePoint.h:326` → `:470` →
`LevelHandler.cpp:294` → `:269`), so the swap ran at a safe point and the
TickTaskManager assertion never fired. This is a **second, independent** fatal on
the same verb, from unclean-teardown rather than from timing: `EditorServer.cpp:2544`,
not `TickTaskManager.cpp:1992`. Deferring to a safe point cannot prevent it.

## Root-cause guess

`FEditorFileUtils::LoadMap` → `Map_Load` → `EditorDestroyWorld` → `Cleanse` →
`CollectGarbage`, then `Map_Load` verifies at `EditorServer.cpp:~2500-2544` that the
outgoing world and its level package were collected, and `appError`s when they were
not. A dirty package holds `GARBAGE_COLLECTION_KEEPFLAGS` (`RF_Standalone`), so it
survives the collect — which the log states verbatim: *"is not currently reachable
but it does have some of GARBAGE_COLLECTION_KEEPFLAGS set"*.

Interactively the engine's own guard is the modal *"The following assets have been
modified and cannot be unloaded"* dialog, whose purpose is to stop the caller. Under
PinWright's unattended operation that dialog is auto-answered `Ok`
(`Message dialog closed, result: Ok` in the log) and the load proceeds into the
condition the dialog exists to prevent. So unattended mode converts a user-refusable
state into a process kill. Note the leaked world here is the **target** map, not the
outgoing one — `FPS_Compound` was resident-and-dirty from an earlier session while
`T_AI` was the active world.

Not read in source; frames and line numbers above are from the crash log, and
`LevelHandler.cpp:269` / `:294` are the PinWright call sites it names.

## Suggested fix

Before `FEditorFileUtils::LoadMap`, check every dirty world package
(`EditorLoadingAndSavingUtils::GetDirtyMapPackages`, or `UPackage::IsDirty` on the
target world's package when it resolves in memory) and:

- refuse with a typed error naming the dirty map(s) and the remedy (`level.save`,
  or an explicit `discardDirty: true`), **or**
- accept an explicit opt-in argument that saves or force-cleans them first.

Either is strictly better than the current behaviour, which is a fatal in a shared
process. The same guard belongs on `editor.open_level` and `editor.open_asset`
(World), which cross-dispatch into this call site.

## Workaround

None from inside the editor once it is gone. Preventatively: never `level.load` a
map while any world package is dirty — call `editor.list_dirty_packages` first and
save or discard. In a shared editor that is not sufficient either, since another
agent can dirty a map between the check and the load.

## History

- `#1-filed` `OPEN` reporter — Hit while starting the FPS VFX stream on UE 5.8 / EAContentExamples58 in a six-agent shared editor. My own calls (`asset.list`, `niagara.inspect`, `material.decompile_mgir`) had been succeeding for ~8 minutes; another stream then issued `level.load` on `/Game/FPS/Maps/FPS_Compound` and the process died between one of my calls and the next, with every subsequent `call()` returning `EDITOR_NOT_RUNNING ... (connection refused)`. The log shows the whole chain in eight lines: `MAP LOAD` issued, `T_AI` cleaned up, the engine's *"modified and cannot be unloaded"* modal auto-answered `Ok` for `/Game/FPS/Maps/FPS_Compound`, a reference-chain dump proving the surviving world is held only by `GARBAGE_COLLECTION_KEEPFLAGS` (i.e. it is dirty), two `LogEditorServer: Error: ... not cleaned up by garbage collection while loading new map!` lines, then `appError` at `EditorServer.cpp:2544`. Filed as distinct from `B-open-asset-world-map-load-crash` because the deferred safe-point fix from that ticket's `#6`/`#7` is visibly in the callstack and did its job — the swap ran off-frame from `FTSTicker` — and the process still died, on a different assertion in a different file. Severity Critical: impact = shared-editor process kill taking every attached agent's session with it, reach = any `level.load` in a session where a map package is dirty, which in a multi-agent editor is the normal state rather than the exception. No fix or source read attempted; I do not own the plugin source in this task.
- `#2-dirty-probe-predicts-it` `OPEN` reporter — Second agent hit by the same crash in the same editor (FPS PLAYER stream; my in-flight `python.execute` returned `EDITOR_NOT_RUNNING ... (connection refused)` mid-build). Adds one actionable fact the first report does not have: the crash is **predictable from an existing PinWright verb**. Minutes before the fatal I had called `editor.list_dirty_packages {}` and got exactly `{"count":1,"packages":["/Game/FPS/Maps/FPS_Compound"]}` — the same package the engine later refused to unload and then fataled on. So the guard does not need new engine plumbing: `level.load` can run the same `UPackage::IsDirty` sweep `editor.list_dirty_packages` already runs, and when a **world** package is dirty either refuse with a typed error naming it, or report it in the response (e.g. `outgoingWorldRetained` / `dirtyWorldsResident`) instead of silently auto-answering the engine modal with `Ok` and returning success. Confirming evidence from my session log `X:/src/unreal/EAContentExamples58/Saved/Logs/EAContentExamples58.log:4801-4831`, identical to #1. Workaround now in force for my stream: probe `editor.list_dirty_packages` before every `level.load` and decline the swap while any world package is listed — weak in a shared editor, because the dirty world normally belongs to another agent who alone may save it.
- `#3-caller-side-repro-and-collateral` `OPEN` reporter — I am the agent whose `level.load` fired the fatal (FPS ENV stream, same editor). I had filed this independently as `B-level-load-dirty-resident-world-fatal`; that file is deleted in favour of this one, and the two facts it carried that are not yet here are merged below. **First, how the dirty resident world was created, which is the part neither #1 nor #2 could see:** I ran `level.create {levelPath:"/Game/FPS/Maps/FPS_Compound"}` then `level.save {}` (`{saved:true}`, `.umap` on disk confirmed by `ls`), then made ~20 mutation RPCs against it (`lighting.spawn_light`, `environment.spawn_sky_atmosphere`, `lighting.spawn_sky_light`, `lighting.setup_volumetric_fog`, `lighting.setup_light_shafts`, `post_process.set_lumen_gi` / `set_lumen_reflections` / `set_bloom` / `set_motion_blur` / `set_color_grading`, `actor.set_component_properties` x3, `actor.set_label` x2, `property.set`, and one `python.execute` writing 20 `FPostProcessSettings` fields) without re-saving. **Another agent then swapped the world to `/Game/FPS/Test/T_AI` and I was not told.** The way I found out is worth recording as a second defect surface: my next `actor.spawn {classPath:"/Script/Engine.PlayerStart"}` returned `"mapPath":"/Game/FPS/Test/T_AI"` and spawned my actor **into another stream's map** — `actor.spawn` names the map it wrote to in the response but has no way to require one, so a stream that believes it owns a world silently writes into whichever world is current. I deleted the stray actor immediately; the point is that nothing refused the write. `editor.status` then confirmed `editorWorldPath: /Game/FPS/Test/T_AI`, and my `level.load` back to my own map is what killed the process. So the arming step is not exotic: **save a map, keep editing it, get swapped out, come back.** A single agent reproduces it with no second client at all. **Second, the collateral, which sizes the severity:** every one of those ~20 RPCs was lost — the `.umap` on disk was the empty level from `level.create`, and the lighting rig, fog, sky and the whole post-process volume had to be rebuilt from scratch. Adding to #2's remedy: `editor.list_dirty_packages` predicts it, but the *fix* wanted here is not only a refusal. The requested world was already resident **with the caller's own unsaved edits in it**, so even a `LoadMap` that survived GC would have silently discarded them — reactivating the resident world (`GEditor->SetCurrentWorld`) is the behaviour that is both crash-free and correct, and refusal should be the fallback when the resident world belongs to someone else.
