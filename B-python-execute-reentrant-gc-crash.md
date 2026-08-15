---
id: B-python-execute-reentrant-gc-crash
title: "Editor crash: python.execute handler runs a UFUNCTION that triggers CollectGarbage(), engine re-enters Python from the pre-GC delegate"
status: IN-REVIEW
severity: Critical
category: bug
tags: [python, crash, garbage-collection, engine-fault, reentrancy, material]
encounters: 1
lastSeen: 2026-08-13T02:09:38Z
---

# Editor crash: python.execute handler runs a UFUNCTION that triggers CollectGarbage(), engine re-enters Python from the pre-GC delegate

`system.python_execute` (`AutoHandler_314_`, `PythonExecuteHandler.cpp:172`) crashed UE 5.8 with
`EXCEPTION_ACCESS_VIOLATION reading address 0x00007ff800007383`. This is an **engine fault inside
`PythonScriptPlugin`**, not a PinWright handler defect — but PinWright supplies the calling context
that makes it reachable, so the mitigation options below are ours.

## What actually happened

The script executed by `python.execute` called `unreal.MaterialEditingLibrary.recompile_material()`.
That UFUNCTION calls `FMaterialEditorUtilities::BuildTextureStreamingData()`, which runs a **full
synchronous `CollectGarbage()`** — while a Python frame is still live on the same thread. GC then
broadcasts its pre-collect delegate, which lands in `FPythonScriptPlugin::OnPreGarbageCollect()`,
which re-enters the interpreter that is already mid-call. The re-entrant interpreter access
faults.

Read the stack inside-out — Python calls the engine, the engine calls GC, GC calls Python:

```
UnrealEditor-PinWright.dll!AutoHandler_314_()                       PythonExecuteHandler.cpp:172
UnrealEditor-PythonScriptPlugin.dll!FPythonScriptPlugin::ExecPythonCommandEx()   :894
UnrealEditor-PythonScriptPlugin.dll!FPythonScriptPlugin::RunFile()              :1953
UnrealEditor-PythonScriptPlugin.dll!FPythonScriptPlugin::EvalString()           :1825
  python311.dll!UnknownFunction  (x5)
UnrealEditor-PythonScriptPlugin.dll!FPyMethodWithClosureDef::Call()             PyMethodWithClosure.cpp:170
UnrealEditor-PythonScriptPlugin.dll!FPyWrapperObject::CallClassMethodWithArgs_Impl()  PyWrapperObject.cpp:333
UnrealEditor-PythonScriptPlugin.dll!FPyWrapperObject::CallFunction_Impl()        PyWrapperObject.cpp:314
UnrealEditor-PythonScriptPlugin.dll!PyUtil::InvokeFunctionCall()                PyUtil.cpp:647
UnrealEditor-CoreUObject.dll!UObject::ProcessEvent()                            ScriptCore.cpp:2234
UnrealEditor-CoreUObject.dll!UFunction::Invoke()                                Class.cpp:7595
UnrealEditor-MaterialEditor.dll!UMaterialEditingLibrary::execRecompileMaterial()
UnrealEditor-MaterialEditor.dll!UMaterialEditingLibrary::RecompileMaterial()     MaterialEditingLibrary.cpp:1054
UnrealEditor-MaterialEditor.dll!FMaterialEditorUtilities::BuildTextureStreamingData()  MaterialEditorUtilities.cpp:827
UnrealEditor-CoreUObject.dll!CollectGarbage()                                   GarbageCollection.cpp:6388
UnrealEditor-CoreUObject.dll!UE::GC::FReachabilityAnalysisState::PerformReachabilityAnalysisAndConditionallyPurgeGarbage()  :6007
UnrealEditor-CoreUObject.dll!UE::GC::PreCollectGarbageImpl<1>()                 GarbageCollection.cpp:5741
UnrealEditor-CoreUObject.dll!TMulticastDelegate<...>::Broadcast()
UnrealEditor-PythonScriptPlugin.dll!FPythonScriptPlugin::OnPreGarbageCollect()  PythonScriptPlugin.cpp:2173
  python311.dll!UnknownFunction  (x4)   <-- ACCESS VIOLATION
```

**Aggravating factor that is ours.** Note the frames *below* `AutoHandler_314_`:

```
FRpcDispatcher::ProcessRequest()                 RpcDispatcher.cpp:588
`FRpcDispatcher::DrainAutoRegistrations'::<lambda_1>::operator()()   RpcDispatcher.cpp:333
TGraphTask<FAsyncGraphTask>::ExecuteTask()
FNamedTaskThread::ProcessTasksUntilIdle()
GameThreadWaitForTask()                          RenderingThread.cpp:1223
FRenderCommandFence::Wait()                      RenderingThread.cpp:1267
FFrameEndSync::Sync()                            RenderingThread.cpp:2509
FEngineLoop::Tick()                              LaunchEngineLoop.cpp:6088
```

The handler did **not** run at a clean tick boundary. It ran from a task-graph task that the game
thread pumped *while blocked on the end-of-frame render fence* — a re-entrant point mid-frame, with
rendering commands in flight. The 9 seconds of
`LogRendererCore: Warning: FlushRenderingCommands called recursively! 2 calls on the stack.`
immediately preceding the fault are the same re-entrancy showing up benignly.

## Why the existing guard does not catch it

`RpcDispatcher.cpp:411` defers a request when `UE::IsSavingPackage() || IsGarbageCollecting()` — i.e.
when GC is **already** running at dispatch time. This crash is the opposite direction: GC is idle
when the request starts, and the handler *itself* causes a re-entrant GC part-way through. The
entry-time check cannot see it.

## Trigger conditions

Not "heavy `python.execute` usage" generically. The specific shape is:

> `python.execute` runs a script that calls any UFUNCTION which synchronously invokes
> `CollectGarbage()` while a Python frame is on the stack.

`UMaterialEditingLibrary::RecompileMaterial` is one such function (via `BuildTextureStreamingData`).
Others in the same family are worth auditing — anything calling `CollectGarbage()` /
`GEngine->ForceGarbageCollection(true)` from an editor-scripting UFUNCTION.

Repro sketch (do **not** run casually — it kills the editor):
`python.execute` with `unreal.MaterialEditingLibrary.recompile_material(unreal.load_asset('<some material>'))`.
In this incident the material was `/Game/DotaBlockout/Materials/M_WT_RiverMaster`, which was itself
failing to compile (`(Node Clamp) Missing Clamp input`) — a failing compile plausibly widens the
window by leaving more transient objects for GC to reap, but the fault does not require it.

**Workaround:** avoid calling `recompile_material` (and other GC-forcing UFUNCTIONs) from inside
`python.execute`. Prefer a native PinWright method that performs the recompile outside a live Python
frame — `material.authoring.compile_material` exists and should be checked for whether it takes the
same `RecompileMaterial` path.

## Fix

Candidate mitigations, roughly in order of value:

1. **Do not execute handlers from the frame-end fence re-entrancy point.** The dispatch task is
   being picked up by `GameThreadWaitForTask` inside `FFrameEndSync::Sync`. Marshalling handler
   execution to a real tick callback (the subsystem already runs a 0.1s ticker) instead of letting an
   `FAsyncGraphTask` be pumped from arbitrary game-thread waits would remove a whole class of
   mid-frame re-entrancy, this crash included. Highest-leverage change and it is entirely on our side.
2. **Suppress GC for the duration of a `python.execute` call.** Holding an `FGCScopeGuard` across
   `ExecPythonCommandEx` would make the nested `CollectGarbage()` a no-op / deferred rather than a
   re-entrant collection. Needs care: a game-thread `CollectGarbage()` under a held GC lock may
   assert or deadlock rather than skip, so this must be verified against `GarbageCollection.cpp`
   before adopting. Investigate `GIsGarbageCollecting` / `FGCScopeGuard` semantics on 5.8.
3. **Re-entrancy flag on the Python path.** Set a PinWright-owned "python frame live" flag around
   `ExecPythonCommandEx` and have the dispatcher refuse/defer any nested dispatch while it is set.
   Narrower than (1) but cheap.
4. **Document the hazard.** `system.python_execute`'s wiki page should carry an explicit warning
   naming `recompile_material` and the GC-forcing family, so agents reach for the native method.

Note the answer to "could PinWright avoid holding Python objects across a GC boundary" is: PinWright
holds none — it hands a string to `ExecPythonCommandEx` and reads `FPythonCommandEx` back. The
objects that die are the engine Python plugin's own wrapper objects, inside its own pre-GC callback.
So the fix cannot be about our object lifetimes; it has to be about **not letting a GC start while a
Python frame is live**, which is what (1)/(2)/(3) address.

## History
- `#2-mitigations-1-and-4-landed` `IN-REVIEW` — **(1) and (4) done; (2) rejected with a reason; (3) not
  needed for this trigger.**
  - **(1)** `python.execute` (plus `system.console_command` / `editor.console_command`) added to the
    tick-unsafe table in `Dispatch/SafePoint.cpp`. The dispatcher now re-queues them onto
    `FRpcDispatcher::PendingQueue`, which `UPinWrightSubsystem::Tick` drains from the 0.1 s core
    ticker — exactly the "marshal to a real tick callback instead of an `FAsyncGraphTask` pumped from
    arbitrary game-thread waits" this ticket asked for. **This does not fix THIS crash** and the code
    comment says so: the fault is `CollectGarbage()` running with a live Python frame, a
    stack-CONTENTS problem, and moving the frame to the ticker keeps the frame. It closes the
    separate mid-frame re-entrancy class (`open <map>` -> `FreeTickTaskLevel`) that the same entry
    points could reach.
  - **(2) rejected.** Holding an `FGCScopeGuard` across `ExecPythonCommandEx` converts a crash into a
    same-thread deadlock rather than skipping the collect. Not adopted. **unverified** against
    `GarbageCollection.cpp` on 5.8 — rejected on the shape of the mechanism, not on a read of it.
  - **(4)** done, and it is now a hard refusal rather than prose:
    `Handlers/System/PythonWeakSandbox.h/.cpp` refuses any script containing `recompile_material`
    before `ExecPythonCommandEx` is called, with error code `PYTHON_CALL_BLOCKED` naming
    `material.authoring.compile_material` and carrying `blockedCall` / `replacement` in its data.
    Modelled on the official Blender MCP server's `weak_sandbox.py`, including its inclusion rule
    ("guaranteed to cause problems and/or failure") and its stated reason for not relying on the
    prompt. **Exactly one entry**: a search of this board, the plugin docs, the wiki and every source
    comment found no other call with a recorded incident behind it. `obj gc`,
    `SystemLibrary.collect_garbage`, `EditorAssetLibrary.delete_asset`, the `quit` console command and
    `LevelEditorSubsystem.load_level` were all considered and rejected for want of evidence; two of
    them would have been actively wrong (`load_level` is already handled by the safe point, and
    `editor_request_end_play` is this board's documented recovery path in
    `B-ui-stop-play-fails-active-pie`).
  - The open question in the Workaround above is resolved: `material.authoring.compile_material`
    (`MaterialAuthoringHandler.cpp:2671`) does **not** take the `RecompileMaterial` path — it does
    `PreEditChange`/`PostEditChange`/`MarkPackageDirty` + `MaterialCompileErrorCollector::WaitAndCollect`,
    and a grep for `RecompileMaterial|BuildTextureStreamingData` over the whole plugin source tree
    returns zero matches.
  - Also fixed: `docs/wiki-src/python.md:44` had a dangling "the GC-during-Python crash described
    below" pointing at a section that was never written. It exists now.
  - **Not verified**: no build, no test run (a map agent holds the DLL). Six new tests
    (`PinWright.python.sandbox.*`) plus one added to `PinWright.core.safe_point.*`; none of them ever
    executes the fatal call. Move to `DONE` only after the integration pass builds and the suite runs
    clean.
- `#1-initial-crash-report` `OPEN` reporter — Editor (PID 28004) died with `EXCEPTION_ACCESS_VIOLATION` at
  `FPythonScriptPlugin::OnPreGarbageCollect()` during a `python.execute` call that ran
  `UMaterialEditingLibrary::RecompileMaterial` → `BuildTextureStreamingData` → `CollectGarbage()`.
  Engine Python-plugin fault, reached through PinWright's `system.python_execute` handler, which was
  itself running from a task-graph task pumped inside `FFrameEndSync::Sync` rather than at a tick
  boundary. Full callstack and mitigation candidates above. Log:
  `X:/src/unreal/EAContentExamples58/Saved/Logs/EAContentExamples58-backup-2026.08.13-02.09.38.log`
  lines 6310-6357. No level data lost (map had been saved 10 min earlier; no actor ops in the window).
