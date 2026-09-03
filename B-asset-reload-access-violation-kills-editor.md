---
id: B-asset-reload-access-violation-kills-editor
title: "asset.reload hard-crashes the editor: ReloadPackages faults with EXCEPTION_ACCESS_VIOLATION inside UFunction::Serialize, taking every agent in the shared process with it"
status: OPEN
severity: Critical
category: bug
tags: [asset, asset-reload, reload, editor-crash, access-violation, packagereload, shared-editor, soundcue]
---

# `asset.reload` faults inside `ReloadPackages` and kills the editor process

## Symptom

`asset.reload` on a SoundCue package killed the shared UE 5.8 editor outright. Six agents
lost their sessions; every subsequent `call()` returned
`EDITOR_NOT_RUNNING ... (connection refused)`.

Log, `Saved/Logs/EAContentExamples58.log`, `2026.09.02-20.38.03` .. `20.38.13` UTC:

```
LogUObjectGlobals: Reloading 1 Package(s):
	Asset Name: /Game/FPS/Audio/Cues/SC_Impact_Concrete
LogUObjectHash: Compacting FUObjectHashTables data took 2.14ms
LogStreaming: Display: FlushAsyncLoading(954): 1 QueuedPackages, 0 AsyncPackages
LogWindows: Error: === Critical error: ===
LogWindows: Error: Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0x00000039000dd217
```

Callstack, top frames:

```
FPropertyProxyArchive::operator<<()        PropertyProxyArchive.h:46
UStruct::SerializeExpr()                   ScriptSerialization.inl:243
UStruct::SerializeExpr()                   Class.cpp:2691
UStruct::Serialize()                       Class.cpp:2458
UFunction::Serialize()                     Class.cpp:7608
ReloadPackages()                           PackageReload.cpp:776
UPackageTools::ReloadPackages()            PackageTools.cpp:959
UnrealEditor-PinWright.dll!AutoHandler_365_ lambda   Handlers/Asset/AssetManageHandler.cpp:1846
TGraphTask<FAsyncGraphTask>::ExecuteTask()
FNamedTaskThread::ProcessTasksNamedThread()
UMassEntityEditorSubsystem::Tick()         MassEntityEditorSubsystem.cpp:196
FTickableObjectBase::SimpleTickObjects()
UEditorEngine::Tick()                      EditorEngine.cpp:1936
```

There is a ~9.4 s gap between the reload starting (`20:38:03.677`) and the fault
(`20:38:13.102`), filled by an unrelated audio-device timeout — so the reload was in flight
for a long time before it died, rather than faulting immediately.

## Reading the callstack

Two things stand out and both look load-bearing:

1. **It faults serialising a `UFunction`'s bytecode**, not asset data —
   `UFunction::Serialize` -> `SerializeExpr` -> `FPropertyProxyArchive::operator<<`. That is
   the script-expression path, which walks `FProperty*` pointers. Reading
   `0x00000039000dd217` is a garbage pointer rather than a null one, which reads as a
   stale/freed `FProperty` reached through a proxy archive during the reload's fixup —
   i.e. something still referenced the old package's function objects when the reload
   swapped them out.
2. **PinWright runs this off the game thread.** The handler lambda executes inside
   `TGraphTask<FAsyncGraphTask>::ExecuteTask` on a named task thread, reached from
   `UMassEntityEditorSubsystem::Tick` draining the task graph. `UPackageTools::ReloadPackages`
   is editor code that tears down and re-creates `UObject`s; running it from a task-graph
   drain rather than at a clean game-thread point is the same class of position hazard that
   `B-open-asset-world-map-load-crash` fixed for `level.load` with `MapSwapSafePoint`. That
   fix's own header documents the pattern: a heavy destroy-and-recreate must not run while
   the engine is mid-tick.

Both readings are inference from the callstack; I did not read the plugin source.

## Suggested fix

- Route `asset.reload` through the same safe point `level.load` now uses
  (`Handlers/Level/MapSwapSafePoint.h`), so `ReloadPackages` runs from a clean core-ticker
  pass rather than from inside a tick's task-graph drain.
- Refuse, or at minimum warn, when the target package is referenced by a loaded Blueprint
  or by any object with live `UFunction` bytecode — the fault is in function serialisation,
  so a referenced-function check is the specific guard.
- Whatever the outcome, the verb should not be able to take the process down: wrap the
  reload so a failure is an RPC error, not an unhandled exception.

## Workaround

Do not call `asset.reload` in a shared editor. If a package must be re-read from disk,
prefer re-opening or re-resolving the asset. There is no in-band way to know the call is
about to fault.

## History

- `#1-filed` `OPEN` reporter — Third distinct editor-killing crash on this host in one session, and the second with a PinWright frame directly above the engine fault (the others: `B-level-load-dirty-world-memory-leak-fatal` / `B-level-load-dirty-world-fatal`, and a separate access violation painting a Widget Blueprint). I did not make this call — the AUDIO stream reloaded `/Game/FPS/Audio/Cues/SC_Impact_Concrete` while I was mid-capture in another map; my previous RPC at `20:37:52` was a `render.capture_open_level` that succeeded, and the next call after the reload returned `EDITOR_NOT_RUNNING`. Filed from the log rather than from my own repro, so the trigger conditions on the audio side are not characterised here and the owning stream should add what it was doing. What is solid: the verb, the asset, the engine fault, and the off-game-thread execution position, all quoted above. Severity Critical: impact = shared-editor process kill destroying every attached agent's unsaved work (this session has now lost a full wave of Niagara systems to one such crash) x reach = any `asset.reload` call.
- `#2-owning-stream-repro-conditions` `OPEN` reporter — I am the audio stream that made this call, adding the trigger conditions `#1` asked for. Exact call: `asset.reload {"assetPath": "/Game/FPS/Audio/Cues/SC_Impact_Concrete"}`, issued as the first of four `asset.reload` calls sent in one batch (the other three targeted `SC_Impact_Generic`, `SC_Step_Generic`, `SC_Step_Water`); only the first reached the editor — it returned `Editor stream at http://127.0.0.1:27145/mcp ended early (stream read failed: [WinError 10054] ...)` and the remaining three returned `EDITOR_NOT_RUNNING ... (connection refused)`. The log confirms the batch was not the trigger: `LogUObjectGlobals: Reloading 1 Package(s)` — a single package was in flight when it faulted, so concurrency is not required to reproduce. State of the target at reload time, all of it authored in this session and all of it saved to disk before the call: `SC_Impact_Concrete` is a `USoundCue` created by `create_sound_cue`, given a `SoundNodeModulator_0` -> `SoundNodeRandom_0` -> two `SoundNodeWavePlayer` node tree via `add_cue_node` x4 + `connect_cue_nodes` x3, its `FirstNode` set by `property.set` (there is no verb for it — see `E-cue-graph-verbs-cannot-set-firstnode`), `SoundClassObject` and modulator Pitch/Volume min-max set from `python.execute`, `AttenuationSettings` set by `set_cue_attenuation`, then force-saved via `EditorAssetLibrary.save_asset(only_if_is_dirty=False)` (log line `SAVE /Game/FPS/Audio/Cues/SC_Impact_Concrete -> True` at `20:36:55`). Crucially for the `UFunction::Serialize` frame in `#1`: **at reload time the cue was referenced by a loaded Blueprint-generated class** — I had, 43 seconds earlier at `20:37:20`, repointed `/Game/FPS/Audio/DA_ImpactSFX` (class `BP_DA_ImpactSFX_C`, a Blueprint DataAsset) so that its `ImpactSounds` map key 1 holds this exact cue, and saved it. That matches `#1`'s reading exactly — the reload's referencer fixup walked a live Blueprint class's function bytecode — and it makes the suggested "refuse when the target is referenced by a loaded Blueprint" guard the specific fix for this repro. Crash artefacts for this instance: `Saved/Crashes/UECC-Windows-8B28A7474CFEC6ECA4C07482C283751E_0003` (`CrashType=Crash`, `EXCEPTION_ACCESS_VIOLATION reading address 0x00000039000dd217`), same callstack as quoted in the body. I confirm the workaround as written: my own on-disk verification had to fall back to `ls`/`grep -a` over the `.uasset` bytes, because the one verb that proves a saved asset reloads to the same shape is the verb that kills the process. Severity unchanged.
- `#3-verb-is-not-on-the-tick-unsafe-list` `OPEN` reporter — Third stream hit by this same crash instance (weapons/mesh; I lost a `model.compile` issued at `20:38:48`, and the `.uasset` on disk was left one revision behind its committed `.pwmodel` source until the editor came back). I am not re-reporting the outage — `#1` and `#2` have the trigger and the referencer analysis. What I am adding is a **second, independent defect on the same call path that neither entry names, and that the proposed Blueprint-referencer guard would not close**: `asset.reload` is not tick-gated at all. `PinWrightSafePoint::IsTickUnsafeMethod` (`Dispatch/SafePoint.cpp:432`) is a lookup into `GTickUnsafeMethodNames`; that array holds **38** verbs and the only `asset.*` member of it is `asset.generate_thumbnail`. So the dispatcher's `IsTickUnsafeMethod(Method) && !IsSafeNow()` deferral (`Dispatch/RpcDispatcher.cpp:590`) is never even consulted for this verb, and the handler runs wherever the request happens to be drained. That is precisely the position the `#1` callstack records: `TGraphTask<FAsyncGraphTask>::ExecuteTask` -> `FNamedTaskThread::ProcessTasksNamedThread` -> `UE::Tasks::Private::TryWaitOnNamedThread` -> `UMassEntityEditorSubsystem::Tick` -> `FTickableObjectBase::SimpleTickObjects` -> `UEditorEngine::Tick` — i.e. `UPackageTools::ReloadPackages` evicting and re-serializing a live package **re-entrantly, from inside another editor subsystem's Tick**, while that subsystem is parked in a task wait and pumping the named-thread queue on its own stack. Package eviction is a between-frames operation by construction; running it inside a frame that already holds live `UObject` pointers is a fault waiting for a referencer, and the Blueprint-bytecode walk `#2` characterises is one such referencer rather than the whole cause. NOTE THE INTERACTION WITH `B-safepoint-tick-gate-inert-on-simpletickobjects-path`, which is marked `DONE`: that ticket fixed the gate for the `SimpleTickObjects` stack, and this crash ran that exact stack anyway — because a gate cannot defer a verb that was never put on the list. Whatever that fix does for the 38 listed verbs, it does nothing here. Minimum remedy, independent of any referencer guard: add `TEXT("asset.reload")` to `GTickUnsafeMethodNames` so the existing deferral machinery moves it to a safe point, and audit the rest of the `asset.*` namespace for the same omission — `asset.reload` is the verb whose entire job is to evict and re-serialize a package, and it is less gated than `asset.generate_thumbnail`. Status left `OPEN`; severity unchanged at Critical.
