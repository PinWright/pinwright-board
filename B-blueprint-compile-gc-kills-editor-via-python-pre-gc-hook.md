---
id: B-blueprint-compile-gc-kills-editor-via-python-pre-gc-hook
title: "blueprint.set_default / blueprint.compile kill the whole editor: CompileSynchronouslyImpl forces CollectGarbage(), whose pre-GC broadcast faults inside FPythonScriptPlugin::OnPreGarbageCollect — with NO Python frame on the stack, 8 s after another agent's python.execute returned"
status: IN-REVIEW
severity: Critical
category: bug
tags: [blueprint, compile, set_default, crash, garbage-collection, python, engine-fault, cross-agent, shared-editor, weapons]
encounters: 1
lastSeen: 2026-09-06T06:03:38Z
---

# A blueprint compile is a GC-forcing call, and that is enough to detonate a poisoned Python plugin

`blueprint.set_default` on `/Game/FPS/Weapons/BP_WeaponBase` (`PenetrationThickness` 16 -> 28) killed
editor PID 30516 outright: `EXCEPTION_ACCESS_VIOLATION` **writing** address `0x00007ffd00007383`,
four frames deep in `python311.dll`, entered from `FPythonScriptPlugin::OnPreGarbageCollect()`.

**No Python was in the calling path.** The crashing RPC is a blueprint CDO write. The GC that reached
the Python plugin was forced by `FBlueprintCompilationManagerImpl::CompileSynchronouslyImpl`, which
calls `CollectGarbage()` at `BlueprintCompilationManager.cpp:426`.

## Callstack, outside-in — a blueprint compile calls GC, GC calls Python

```
UnrealEditor_PinWright!UPinWrightSubsystem::Tick()                          PinWrightSubsystem.cpp:509
UnrealEditor_PinWright!FRpcDispatcher::ProcessPendingRequests()             RpcDispatcher.cpp:1159
UnrealEditor_PinWright!FRpcDispatcher::ProcessRequest()                     RpcDispatcher.cpp:955
UnrealEditor_PinWright!AutoHandler_342_()                                   BlueprintPropertyHandler.cpp:522
UnrealEditor_PinWright!BlueprintHandlerUtils::CompileBlueprintWithDiagnostics()  BlueprintHandlerUtils.cpp:491
UnrealEditor_UnrealEd!FKismetEditorUtilities::CompileBlueprint()            Kismet2.cpp:807
UnrealEditor_Kismet!FBlueprintCompilationManagerImpl::CompileSynchronouslyImpl()  BlueprintCompilationManager.cpp:426
UnrealEditor_CoreUObject!CollectGarbage()                                   GarbageCollection.cpp:6388
UnrealEditor_CoreUObject!UE::GC::FReachabilityAnalysisState::PerformReachabilityAnalysisAndConditionallyPurgeGarbage()  :6007
UnrealEditor_CoreUObject!UE::GC::PreCollectGarbageImpl<1>()                 GarbageCollection.cpp:5741
UnrealEditor_CoreUObject!TMulticastDelegate<void()>::Broadcast()            DelegateSignatureImpl.inl:1135
UnrealEditor_PythonScriptPlugin!FPythonScriptPlugin::OnPreGarbageCollect()  PythonScriptPlugin.cpp:2178
  python311  (x4)   <-- ACCESS VIOLATION (writing 0x00007ffd00007383)
```

Note the dispatcher frames: this ran from `UPinWrightSubsystem::Tick` off the core ticker, i.e. from
the **clean safe point**, not from the `FFrameEndSync` re-entrancy point that
`B-python-execute-reentrant-gc-crash` records. The safe-point hop does not help here.

## The measurement that makes this a different defect

Log `Saved/Crashes/UECC-Windows-8BF3A2E34A383B2A4D865AA84C88F41C_0002/EAContentExamples58.log`,
lines 3422-3523 (the file ends at the crash):

```
[2026.09.06-06.03.15:161][147] LogPinWrightSafePoint: Running 'python.execute' (id=23bb2c81-...) inline
[2026.09.06-06.03.30:494][193] LogPinWrightSafePoint: Running 'python.execute' (id=3e89e9ec-...) inline
[2026.09.06-06.03.30:658][193] LogPython: [wpn_dump] SM_WPN_Pistol_Slide: wrote 2186 triangles -> ...
[2026.09.06-06.03.38:495][217] LogPinWrightSafePoint: Running 'blueprint.set_default' (id=88cf0cb6-...) inline
[2026.09.06-06.03.38:500][217] LogBlueprint: Compiling Blueprint '/Game/FPS/Weapons/BP_WeaponBase.BP_WeaponBase'
<end of log>
```

Both `python.execute` calls belonged to **another agent** sharing this editor (a geometry-script
triangle dump over `SM_WPN_*`). Both had **returned**: the second one's own results are printed in
full at `06:03:30:658` and its RPC response was delivered. Eight seconds later a different caller's
blueprint compile faulted in the Python plugin's pre-GC hook.

So the pre-condition is not "a Python frame is live on the stack". It is **"a `python.execute` ran
earlier in this editor session"** — the plugin's wrapper state is left in a state its own pre-GC
callback cannot walk, and it stays that way after the script returns. The next synchronous
`CollectGarbage()` from *any* source is the trigger, and a blueprint compile is one.

## Why this is not covered by `B-python-execute-reentrant-gc-crash`

That ticket's mechanism, title, trigger conditions and workaround are all scoped to a live Python
frame:

- its title names `python.execute` as the crashing verb; here the crashing verb is `blueprint.set_default`;
- its diagnosis is "GC re-enters the interpreter **that is already mid-call**"; here the interpreter
  was idle and had been for 8 s;
- its workaround — "avoid calling `recompile_material` (and other GC-forcing UFUNCTIONs) from
  **inside** `python.execute`" — was followed to the letter by every party in this incident and did
  not prevent the crash;
- its mitigation candidates (2) and (3), an `FGCScopeGuard` around `ExecPythonCommandEx` and a
  "python frame live" re-entrancy flag, are both scoped to the duration of the Python call and
  therefore cannot see this one.

Same fault site, same corrupted-pointer shape (`...00007383` in both), different trigger, much larger
blast radius. Recorded separately so the blast radius is not lost under a `python.execute` title.

## Blast radius — this is a shared-editor defect, not a caller's problem

The crash kills the editor for **every** agent in it. Three workstreams shared PID 30516 here: the
Python caller, a capture/PIE stream (`render.capture_open_level` / `effect.advance_simulation` at
05:58), and this one. Nothing warns any of them. The verb that detonates it is one of the most
ordinary in the plugin — every `blueprint.compile`, `blueprint.set_default`, `blueprint.compile_bpir`
and `blueprint.add_variable` route reaches `FKismetEditorUtilities::CompileBlueprint`, hence
`CompileSynchronouslyImpl`, hence `CollectGarbage()`.

There is no way for a blueprint-editing agent to know a sibling ran Python.

Losses were small only by luck: `level.save` had run at 05:58:43, five minutes earlier, and this
agent's material write (`M_WPN_OpticLens`) had been force-saved to disk at `08:57:50 +0300` before
the compile. The blueprint edit itself did not land — `BP_WeaponBase.uasset` mtime is unchanged at
`2026-09-06 00:41:35 +0300`.

## Environment

- UE `5.8.2-56702186+++UE5+Release-5.8`, PID 30516, `SecondsSinceStart` 806.
- Command line `-AutoDeclinePackageRecovery -RunningUnattendedScript`.
- Editor world at crash time `/Game/FPS/Test/T_VFX.T_VFX`; PIE not running (`editor.pie_status`
  polled `inPie: false` immediately before the call, as the compile guard requires).
- Crash dir: `X:/src/unreal/EAContentExamples58/Saved/Crashes/UECC-Windows-8BF3A2E34A383B2A4D865AA84C88F41C_0002/`.

## Fix

Verdict: **PARTLY TRUE** (`valid-bug-wrong-fix`). The callstack proves that PinWright's shared full
Blueprint compile path reached compile-end `CollectGarbage()`, which broadcast pre-GC into
`FPythonScriptPlugin::OnPreGarbageCollect` and crashed the editor. It does not prove the broader
claim that any completed `python.execute` permanently poisons the editor session, so no session
flag, response warning, or documentation contract was added for that theory.

`BlueprintHandlerUtils::CompileBlueprintWithDiagnostics` now passes
`EBlueprintCompileOptions::SkipGarbageCollection` at the sole full-compile choke point. Compilation,
diagnostics, reinstancing, and Blueprint broadcasts remain active, but the handler no longer forces
the compile-end GC that entered the Python pre-GC hook. It immediately requests an equivalent full
purge with `GEngine->ForceGarbageCollection(true)`, so cleanup is deferred to the next engine GC
opportunity after the handler stack unwinds. That later collection still invokes PythonScriptPlugin's
pre-GC callback; this changes the observed call-stack ordering but does not prove the broader
persistent-Python-corruption theory safe. Existing safe-point entries remain because reinstancing is
independently unsafe during ticker re-entrancy.

`PinWright.blueprint.compile.FullCompileRoutesSurvivePythonThenLaterGarbageCollection` first invokes
real `python.execute`, then real `blueprint.compile` and `blueprint.set_default` handlers against a
transient fixture. It uses named skips when the Python module or interpreter is unavailable, requires
zero synchronous pre-GC broadcasts during the handlers, explicitly collects afterward, and requires
exactly one pre-GC callback and zero handled ensures. The infrastructure ratchet independently
requires exactly one `SkipGarbageCollection` compile option and one deferred full-GC request. This is
source-only rework: compile, automation, editor, and crash-reproduction proof remain unverified. The
deferred collection also runs outside `GIsGCingAfterBlueprintCompile` and may remain pending in a
commandlet that never reaches an engine GC opportunity.

## History
- `#1-initial-crash-report` `OPEN` reporter — Editor PID 30516 killed by `blueprint.set_default` on
  `/Game/FPS/Weapons/BP_WeaponBase`. `EXCEPTION_ACCESS_VIOLATION writing 0x00007ffd00007383` in
  `python311` under `FPythonScriptPlugin::OnPreGarbageCollect`, reached from
  `CompileSynchronouslyImpl` -> `CollectGarbage()`. No Python frame on the stack; the nearest
  `python.execute` was another agent's, completed 8 s earlier with its results logged in full. Full
  stack, log excerpt and crash-dir path above. The blueprint edit did not land, and the task it
  belonged to (raising `PenetrationThickness` 16 -> 28 to close the pistol's ~1.12 deg penetration
  incidence margin against the 10 uu `Pen_WoodPanel`) is blocked until the editor is back up.
- `#2-bystander-fourth-stream` `OPEN` reporter — ENV was a **fourth** stream in PID 30516, not one of
  the three counted above, and it lost the whole build-07 slot to this. At 06:00-06:02Z ENV had
  `model.validate`d two new meshes clean and read a material instance; the next call,
  `editor.pie_status` at 06:04Z, returned `EDITOR_NOT_RUNNING` and port 27145 was closed with no
  `UnrealEditor*` process left. Nothing ENV called was a blueprint verb or `python.execute`, and ENV
  had no way to know either had run. Two consequences for candidate (2): the warning belongs on
  **every** response in a session where `python.execute` has run, not only on compile-route ones,
  because the streams that lose their work are the ones that never call the detonating verb; and a
  stream that is only reading (`model.validate` creates nothing) still has to re-establish the whole
  editor to continue, so "informational" understates it. Also worth recording that the crash landed
  while another stream held an active PIE session (`editor.status` at 05:59Z: `inPie: true`,
  `pieIsPaused: true`, `T_Weapons`) — the compile guard's `pie_status` probe was truthful for the
  caller's own map and blind to the sibling's.
- `#3-skip-blueprint-compile-gc` `IN-REVIEW` developer — Changed `BlueprintHandlerUtils::CompileBlueprintWithDiagnostics` to pass `EBlueprintCompileOptions::SkipGarbageCollection`, so every PinWright full Blueprint compile still compiles and reinstantiates but no longer forces the compile-end `CollectGarbage()` that entered `FPythonScriptPlugin::OnPreGarbageCollect`. Added handler-level pre-GC counter coverage for `blueprint.compile` and `blueprint.set_default`, and updated the compile-route ratchet. Kept the existing safe-point gate because reinstancing remains independently tick-unsafe. The broader claim that any completed `python.execute` permanently poisons the session remains unverified.
- `#4-verifier-returned-deferred-gc-and-python-repro` `OPEN` tester — Returned the submission because
  removing immediate compile-end GC provided no replacement cleanup, and the regression never ran
  `python.execute` or a later explicit GC. Engine source does not support the proposed PinWright
  GIL-imbalance diagnosis: Python execution and the pre-GC callback already own balanced private
  scoped GILs. Rework requires deferred full GC, a real Python/compile/explicit-GC regression, and a
  structural ratchet for both source invariants.
- `#5-deferred-gc-and-python-repro` `IN-REVIEW` developer — Retained
  `SkipGarbageCollection` and added the guarded deferred `ForceGarbageCollection(true)` request at
  the shared compile helper. Expanded the handler-level regression to run real `python.execute`,
  `blueprint.compile`, and `blueprint.set_default`, with named Python-availability skips, zero
  synchronous pre-GC broadcasts, one later explicit pre-GC broadcast, and zero handled ensures.
  Updated the structural ratchet to pin both compile-helper invariants and corrected the wiki's
  cleanup and Python-callback claims. No compile, automation, editor, or runtime proof was run.
