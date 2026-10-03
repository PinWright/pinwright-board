---
id: B-python-execute-reentrant-gc-crash
title: "Editor crash: python.execute handler runs a UFUNCTION that triggers CollectGarbage(), engine re-enters Python from the pre-GC delegate"
status: IN-REVIEW
severity: Medium
category: bug
tags: [python, crash, garbage-collection, engine-fault, reentrancy, material]
encounters: 3
costly: 2
lastSeen: 2026-09-06T06:03:44Z
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
- `#3-blocklist-removed-back-to-open` `OPEN` — **The blocklist described in `#2` was built and then
  removed at the user's request before it ever shipped. Back to `OPEN`: this crash has no code-level
  mitigation. `#2` is kept below as the record of what was tried, but its mitigation-(4) claim no
  longer holds.**

  Removed: `Handlers/System/PythonWeakSandbox.{h,cpp}`, `Tests/Utility/TestPythonWeakSandbox.cpp`,
  `ERR_PYTHON_CALL_BLOCKED`, and the pre-flight call site in `PythonExecuteHandler.cpp` (that file is
  now byte-identical to `b92ba268`). Nothing refuses `recompile_material` today.

  **Why it was rejected — do not re-propose this without reading it.** The objection was not that the
  implementation was wrong; it was that a new enforcement mechanism was being added to the
  most-called verb in the plugin to restate a rule that already existed in prose, and nobody had
  asked for it. The Blender-MCP `weak_sandbox.py` precedent that motivated it is a real precedent,
  but "another MCP server does this" is not a reason this plugin needs it. Cost was permanent
  (a scan on every `python.execute`, a mechanism to maintain and a false-positive surface);
  benefit was one call that documentation already covers.

  **What replaced it: documentation, treated as the deliverable rather than as a consolation.**
  `docs/wiki-src/python.md` now carries the crash chain, the freeze hazard, and the working
  alternatives, cross-referenced from `system.md` and `editor.md`. It is written as "this will break
  your editor and here is what to do instead", never as "the plugin protects you", because it does
  not.

  Kept from `#2`, because it was a separate task (#123) and stands on its own: the safe-point gating
  of `python.execute` / `system.console_command` / `editor.console_command` in
  `Dispatch/SafePoint.cpp`, and its tests. That gating does **not** fix this ticket, and the code
  comment now says so explicitly rather than pointing at the deleted sandbox header.

- `#2-mitigations-1-and-4-landed` `IN-REVIEW` — **SUPERSEDED BY `#3`: the mitigation-(4) blocklist
  described here was removed before shipping. Retained as the record of an approach that was tried
  and rejected.** Original entry follows.

  **(1) and (4) done; (2) rejected with a reason; (3) not needed for this trigger.**
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
- `#4-review-a-current-path` `OPEN` reporter — Additional evidence: **Adversarial review A — current crash path remains unmitigated.** Actuality: CONFIRMED CURRENT. Framing: the title and Critical severity remain accurate for the confirmed `recompile_material` editor-crash path, but the body should distinguish that trigger from other GC-forcing calls and remove the already-landed safe-point hop as a fix for this crash. Proposed fix: INCOMPLETE, because safe-point gating changes stack position only; UE 5.8's `FGCScopeGuard` skips locking on the game thread, and a dispatcher re-entrancy flag cannot observe an engine UFUNCTION's nested `CollectGarbage()`. Evidence: `X:/src/unreal/unreal-fpv-dev/Plugins/PinWright/Source/PinWright/Private/Handlers/System/PythonExecuteHandler.cpp:172` `C:/UE_5.8/Engine/Source/Editor/MaterialEditor/Private/MaterialEditingLibrary.cpp:1052` `C:/UE_5.8/Engine/Source/Editor/MaterialEditor/Private/MaterialEditorUtilities.cpp:825` `C:/UE_5.8/Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp:5741` `C:/UE_5.8/Engine/Plugins/Experimental/PythonScriptPlugin/Source/PythonScriptPlugin/Private/PythonScriptPlugin.cpp:2170` `C:/UE_5.8/Engine/Plugins/Experimental/PythonScriptPlugin/Source/PythonScriptPlugin/Private/PythonScriptPlugin.cpp:1823` `C:/UE_5.8/Engine/Plugins/Experimental/PythonScriptPlugin/Source/PythonScriptPlugin/Private/PyUtil.cpp:1824` `C:/UE_5.8/Engine/Source/Runtime/CoreUObject/Private/UObject/GCScopeLock.h:52` `X:/src/unreal/unreal-fpv-dev/Plugins/PinWright/Source/PinWright/Private/Dispatch/SafePoint.cpp:122` `X:/src/unreal/unreal-fpv-dev/Plugins/PinWright/Docs/wiki-src/python.md:58`. Runtime: NOT VERIFIED (no Unreal/build/reproduction run; the prior crash record remains historical evidence). Recommendation: REFRAME; retain OPEN Critical, narrow guaranteed scope to `recompile_material`, and pursue an engine-boundary mitigation that skips or defers Python wrapper GC while `GIsRunningUserScript` is true, or obtain explicit approval for a targeted refusal/native replacement; add an integration regression test before any DONE transition.
- `#5-additional-research-b` `OPEN` reporter — Additional evidence: **Adversarial review B — agrees with A that the synchronous `recompile_material` chain remains unmitigated and that safe-point routing changes stack position, not the live Python frame; disagrees that narrowing the scope to `python.execute`/one call is sufficient.** Actuality: PARTIAL (current source still exposes the path; runtime NOT VERIFIED). Framing: the historical title and Critical severity are justified, but the affected surface is broader: `recorder.query` embeds an unrestricted caller body and calls `ExecPythonCommandEx` too, so the same engine boundary is reachable there; conversely, `SystemLibrary.collect_garbage` is only `ForceGarbageCollection(true)` and queues work for `ConditionalCollectGarbage`, not a synchronous nested collect. Proposed fix: INCOMPLETE, because docs/native replacement and the existing safe-point gate do not prevent direct nested GC; a text blocklist is a band-aid (and was explicitly removed), while `FGCScopeGuard` cannot suppress game-thread GC. Evidence: `X:/src/unreal/unreal-fpv-dev/Plugins/PinWright/Source/PinWright/Private/Handlers/System/PythonExecuteHandler.cpp:172` `X:/src/unreal/unreal-fpv-dev/Plugins/PinWright/Source/PinWright/Private/Handlers/Recorder/RecorderQueryHandler.cpp:64-104` `X:/src/unreal/unreal-fpv-dev/Plugins/PinWright/Source/PinWright/Private/Handlers/Recorder/RecorderQueryHandler.cpp:139-155` `X:/src/unreal/unreal-fpv-dev/Plugins/PinWright/Source/PinWright/Private/Handlers/Recorder/RecorderQueryHandler.cpp:239` `C:/UE_5.8/Engine/Source/Editor/MaterialEditor/Private/MaterialEditorUtilities.cpp:825` `C:/UE_5.8/Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp:6366-6388` `C:/UE_5.8/Engine/Plugins/Experimental/PythonScriptPlugin/Source/PythonScriptPlugin/Private/PythonScriptPlugin.cpp:1823,2170` `C:/UE_5.8/Engine/Source/Runtime/Engine/Private/KismetSystemLibrary.cpp:2885-2888` `C:/UE_5.8/Engine/Source/Runtime/Engine/Private/UnrealEngine.cpp:2136-2140`. Runtime: NOT VERIFIED. Recommendation: KEEP; reframe the ticket around all PinWright Python execution routes plus direct synchronous GC, correct the queued-GC documentation, audit/wrap `recorder.query`, and seek an upstream PythonScriptPlugin guard/defer with an integration regression test before closure.
- `#6-user-reprioritized-medium` `OPEN` reporter — User explicitly reprioritized this Python re-entrant GC crash from Critical to Medium.
- `#7-reached-without-python-in-the-request` `OPEN` reporter — **Second encounter, and it widens the ticket: the caller never touched Python.** Killed the shared FPS editor at 2026-09-06 06:03:44Z, UE 5.8, gateway 27145. The request in flight was `blueprint.set_default` on `/Game/FPS/Weapons/BP_WeaponBase` (WEAPONS stream); the log shows `LogPinWrightSafePoint: Running 'blueprint.set_default' (id=88cf0cb6-4a18-0625-47b7-25a403b9ff4a) inline` at 06:03:38, `LogBlueprint: Compiling Blueprint '/Game/FPS/Weapons/BP_WeaponBase.BP_WeaponBase'` on the same tick, and the fault six seconds later. Callstack, top down: `python311.dll` (4 frames) <- `FPythonScriptPlugin::OnPreGarbageCollect()` <- `TMulticastDelegate::Broadcast()` <- `UE::GC::PreCollectGarbageImpl<1>()` <- `FReachabilityAnalysisState::PerformReachabilityAnalysisAndConditionallyPurgeGarbage` <- `CollectGarbage()` <- `FBlueprintCompilationManagerImpl::CompileSynchronouslyImpl()` <- `FKismetEditorUtilities::CompileBlueprint()` <- `BlueprintHandlerUtils::CompileBlueprintWithDiagnostics()` <- `AutoHandler_342_()` <- `FRpcDispatcher::ProcessRequest()` <- `UPinWrightSubsystem::Tick()`. `EXCEPTION_ACCESS_VIOLATION writing address 0x00007ffd00007383`.
  The faulting delegate and the fault address class are identical to `#1`, so this is the same defect, but the title and every mitigation discussed in `#1`-`#6` are scoped to `python.execute`, and **this path contains no `python.execute` at all** — a Blueprint mutator that compiles synchronously is enough, because `CompileSynchronouslyImpl` calls `CollectGarbage()` itself. Any PinWright verb that can reach a synchronous GC is therefore exposed: `blueprint.set_default`, `blueprint.compile`, every BPIR write (which compile implicitly per PLAN rule 12), and the `recompile_material` chain named in `#4`/`#5`. Gating only `python.execute` would not have prevented this crash.
  Cost: killed the editor with six streams attached. I was the VFX critic holding the world lock two seconds into a capture run; my own `effect.spawn_niagara` had returned success and the following `effect.step_and_capture` came back `EDITOR_NOT_RUNNING (connection refused)` after a 2.1 s settle, so from the caller's side an unrelated stream's Blueprint verb presents as your own capture verb failing. Probed per PLAN rule 6: port 27145 closed AND no `UnrealEditor` process — a full process death, not a wedged game thread. Lock released on token match; no world package was dirty because the world died with the process. Not raising severity: the user set Medium in `#6` and that call is theirs, but that decision was made against a python.execute-only reading of the scope, which `#7` shows is too narrow.

- `#8-the-missing-half-of-encounter-7` `OPEN` reporter — **`#7` says "the caller never touched Python". Correct for that caller — but a DIFFERENT stream's `python.execute` had returned seconds earlier on the same editor, and that is what makes the crash reachable.** WEAPONS build-05 (this reporter) ran `python.execute {mode: execute_file}` against a scratch script that called `GeometryScript_AssetUtils.copy_mesh_from_static_mesh` four times, holding four transient `UDynamicMesh` objects in Python locals, and looped `GeometryScript_MeshQueries.get_triangle_positions` 27,200 times over them. The call **returned success**: the four `[wpn_dump] ... wrote N triangles` lines are the last normal lines in `Saved/Logs/EAContentExamples58.log` before `06.03.38 LogDerivedDataCache`, then `06.03.38 LogPinWrightSafePoint: Running 'blueprint.set_default'`, then the fault at `06.03.44`. So the ordering on the wire is: python.execute completes and its response is delivered -> an unrelated RPC from another stream forces a synchronous `CollectGarbage()` -> `FPythonScriptPlugin::OnPreGarbageCollect()` faults in `python311.dll`.
  **This contradicts the wiki's stated mechanism and should change the wiki text.** `Saved/PinWright/wiki/python.md:81` says "what faults is the Python plugin's *own* pre-GC hook walking its wrapper state **while your Python frame is still live on the same thread**". Here no Python frame was live: the handler had already unwound, restored `sys.modules` per the private-scope contract, and answered. What survives the call is the plugin's UObject<->PyObject wrapper table, and the private-scope teardown appears to leave entries in it that the next `OnPreGarbageCollect` walks after the referent is gone. That reframes the hazard from "do not force a GC inside your script" to **"any `python.execute` that binds UObjects arms the next garbage collection anywhere in the editor, for any stream"** — which is why `#7`'s caller could be innocent, and why `#4`/`#5`'s narrowing to `recompile_material` and to "PinWright Python execution routes" both understate the blast radius. It is a cross-stream landmine on a shared editor: cost here was one editor and every agent in it, with no warning to any of them.
  Correlation, not proof — I did not run a controlled repro, because the only way to try is to kill the shared editor again. What would prove it cheaply: run `python.execute` binding a UObject, then `system.console_command {command: "obj gc"}` on an otherwise idle editor, and see whether the collect faults. Two independent mitigations worth costing: (a) have the handler explicitly drop its Python-side UObject wrappers before answering (the private-scope teardown already snapshots `sys.modules`; the wrapper table is the thing it does not clean), and (b) if that is not reachable, say so on `python.md` and tell callers that a shared editor cannot safely mix `python.execute` with other streams at all.
- `#9-rephrase-python-gc-runs-inside-nested-collect` `OPEN` developer — **REPHRASE (batch 5, against PinWright `7230b41d` + UE 5.8 source).** The title's mechanism, "the engine re-enters Python", is not the defect: re-entry is legal. `PyUtil::InvokeFunctionCall` (`PyUtil.cpp:645`) releases the GIL with `Py_BEGIN_ALLOW_THREADS` around `ProcessEvent`, and `FPythonScriptPlugin::OnPreGarbageCollect` (`PythonScriptPlugin.cpp:2175`) takes it back with `PyGILState_Ensure` on the same thread. What it does next is the hazard: `PyUtil::CollectGarbage()` -> `PyGC_Collect()`, a full cyclic collection of the interpreter heap, which runs mid-script while the caller's frame and its wrappers are live. The two recorded deaths (`#1` here, and `#7` = `B-blueprint-compile-gc-kills-editor-via-python-pre-gc-hook`) both fault inside that collection, with the same fault-address shape (`0x00007ffX00007383`). The `#7`/`#8` path has no Python frame on the stack and is owned by the Blueprint ticket, where synchronous compile GC is already skipped. **Real defect, as scoped here:** any UFUNCTION a `python.execute` script calls that runs `CollectGarbage()` synchronously (for example `recompile_material` via `BuildTextureStreamingData`, or `obj gc` through `SystemLibrary.execute_console_command`) makes the engine run a full Python `gc` pass inside the live script. PinWright can neither defer nor skip the UE collect: there is no engine skip for a game-thread `CollectGarbage()` (`GarbageCollection.cpp:6366`), and `FGCScopeGuard` does not lock on the game thread. It can, however, suspend the interpreter side. `PyGC_Collect()` returns 0 without collecting while `gc.isenabled()` is false (checked on the engine's Python 3.11.8: 0 with gc disabled, 107 with it enabled, on the same cyclic garbage). Not established: what corrupts the GC heap in the first place. No controlled repro; it would kill the shared editor.
- `#10-first-slice-python-gc-suspended-during-script` `IN-REVIEW` developer — **First slice only (E3): the recorded crash path is prevented, and a nested collect is reported in the response.** `Handlers/System/PythonExecuteHandler.cpp`: `RunPython` runs `SuspendGcScript` before the module snapshot. That script pushes `gc.isenabled()` onto `sys._pinwright_gc_states` and calls `gc.disable()`. `ResumeGcScript` runs after the module restore and re-enables gc only if it was enabled before. The stack handles nested calls, and the scripts bind nothing but `sys` in the console globals. A pre-GC delegate counter is registered around `ExecPythonCommandEx` only. When it is above zero, the response's `log` gets a `Warning`. The warning says a synchronous collect ran inside the script and whether the collector was suspended, and it points to `material.authoring.compile_material`. Every mode and scope is covered. Verified offline on the engine's Python 3.11.8: `PyGC_Collect()` returns 0 when gc is disabled and 107 when enabled, nested suspend/resume restores the prior state, and a gc that was already disabled stays disabled. Docs: `docs/wiki-src/python.md` "Deferral cannot fix this" paragraph rewritten. CHANGELOG entry added. Test: `PinWright.python.execute.NestedGarbageCollectionRunsWithPythonGcSuspended` (`Tests/Infra/TestPythonExecuteNestedGcSuspension.cpp`). It runs `obj gc` through `SystemLibrary.execute_console_command` inside the script. It then asserts three things: the warning is present (the precondition that a nested collect ran), the in-script `gc.isenabled()` is `False`, and the value is `True` again after the call. **Not verified:** no build, no run. The test drives a synchronous GC inside a live script, so if the guard fails it could reproduce the crash. **Remaining slices:** (2) put `recorder.query` (`RecorderQueryHandler.cpp`, which also calls `ExecPythonCommandEx` with caller code; see `#5`) under the same suspend/resume. (3) Decide whether a script that calls `gc.enable()` or `gc.collect()` itself should be reported. (4) Find what corrupts the Python GC heap. That needs a controlled repro on a disposable editor, and it is the only slice that could close `#7`/`#8` and `B-blueprint-compile-gc-kills-editor-via-python-pre-gc-hook` for good. (5) Add an integration repro with `recompile_material` on a failing material before DONE.
- `#11-review-fixes-mitigation-wording-cxx-state` `IN-REVIEW` developer — Review fixes.
  - **Slice (2) of `#10` is dropped.** `recorder.query` runs caller code in a bundled-Python child process and never calls `ExecPythonCommandEx`.
  - **Mitigation wording.** CHANGELOG is now `Changed:` and says "mitigates, but does not fix". `python.md` says the pass is moved out of the script, that the same pass has faulted with no script running, and that this is a mitigation. It now also documents the cycle-held-object side effect and the explicit `gc.collect()` workaround. The handler comment cites `#1` only.
  - **Prior state moved to C++.** The `sys._pinwright_gc_states` scripts were replaced by `PythonExecuteGcGuard::IsCyclicGcEnabled` / `SetCyclicGcEnabled`, which are `EvaluateStatement` / `ExecuteStatement` of `__import__('gc')...`. The prior state is a `bool` in the handler's C++ frame. After the script, gc is re-enabled only if it was on before, then read back. A failed restore adds a `Warning` to `log`.
  - **Warning wording.** The GC warning now reads "A synchronous garbage collection ran %d time(s) while this script was running" and names `compile_material` only as the `recompile_material` alternative.
  - **Test precondition.** The test now asserts gc is enabled before the call.
  - **Follow-up.** Remaining slices (3), (4) and (5) moved to the new OPEN ticket `B-python-pre-gc-pass-heap-fault-root-cause`, cross-linked to `B-blueprint-compile-gc-kills-editor-via-python-pre-gc-hook`.
- `#12-linux-verification` `IN-REVIEW` tester — Linux, UE 5.8 Vulkan, PinWright `ae877ccc` on origin/master. Final run b6/run3/full: offscreen full suite, 5827/5827 passed, 0 failed, 73 skipped; the build and Python results come from run2 on the same tree (commit `217d6ddb`). `PinWright.python.execute.NestedGarbageCollectionRunsWithPythonGcSuspended` passed with no skip marker and is not in skipids. The test asserts that gc is on before the call. It runs `obj gc` through `SystemLibrary.execute_console_command` inside the script and asserts the nested-collect `Warning`, which proves a synchronous collect ran. It asserts `gc.isenabled()` is `False` inside the script and `True` after the call. The editor survived that nested collect, and the rest of the suite ran to a clean exit. Run1's end-of-log truncation did not recur in run3, and nothing ties it to this commit (run2/run3 summaries). This is a mitigation, not a fix: the pre-GC `PyGC_Collect` pass moves out of the script but still runs at the next collect. Remains: the `#1`-shape integration repro (`recompile_material` on a failing material, in a disposable editor). `B-python-pre-gc-pass-heap-fault-root-cause` slice 2 names that repro as the gate before this ticket reaches DONE. The heap-corruption root cause and script-side `gc.enable()`/`gc.collect()` reporting are also there. The `recorder.query` slice was dropped because it runs in a child process. A disposable-editor repro or a human decision is needed to close.
