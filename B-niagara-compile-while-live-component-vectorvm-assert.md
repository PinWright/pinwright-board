---
id: B-niagara-compile-while-live-component-vectorvm-assert
title: "compile:true on any niagara.* edit recompiles a system whose ANiagaraActor is still ticking, and the live emitter instance executes the new bytecode against its old DataSets: VectorVM asserts on a worker thread and the editor dies"
status: IN-REVIEW
severity: Critical
category: bug
tags: [niagara, compile, editor-crash, assertion, vectorvm, live-instance, kill-system-instances, worker-thread, shared-editor]
encounters: 2
lastSeen: 2026-08-27T20:50:00+05:00
---

# `compile: true` does not kill the live system instances first, so the simulation asserts inside the VectorVM

Every `niagara.*` edit verb takes `compile`, and the shared finalizer calls
`UNiagaraSystem::RequestCompile(true)` **without first destroying the running
`FNiagaraSystemInstance`s of that system**. If an `ANiagaraActor` in the open level is
playing the system, its emitter instance keeps ticking on a task-graph worker while the
scripts are swapped underneath it, executes the new bytecode against the old data-set
layout, and hits

```
Assertion failed: DataSetIdx < ExecCtx->DataSets.Num()
[File:D:\build\++UE5\Sync\Engine\Source\Runtime\VectorVM\Private\VectorVMRuntime.cpp] [Line: 421]
```

That is an `appError` on a worker thread, so the whole editor process dies and takes
every other agent's unsaved work with it.

The plugin already has the exact helper needed —
`PinWrightNiagara::KillSystemInstances` — and four sibling handlers already call it. The
shared compile path is the one that does not.

## Repro

Authoring a system while previewing it in the level, which is the workflow the
`niagara` docs recommend (`niagara.spawn_actor` + `effect.activate_niagara` +
`render.capture_open_level`, the standing workaround for the
`render.capture_asset_preview` crashes):

```
niagara.spawn_actor      {systemPath: "/Game/Atlantis/VFX/NS_FishSchool",
                          location: {x:19500, y:-7000, z:2000}, name: "PWFISH_Probe"}
effect.activate_niagara  {systemName: "PWFISH_Probe", reset: true}
// ... leave it running and keep authoring, as you must in order to see your edits ...
niagara.set_property     {assetPath: "/Game/Atlantis/VFX/NS_FishSchool",
                          target: {kind:"renderer", emitter:"Fountain", index:0},
                          propertyPath: "MeshIndexBinding.BindingSourceMode",
                          value: "ImplicitFromSource",
                          compile: true, save: true}
```

The call never returns — the transport reports
`stream read failed: [WinError 10054]` — and the process is gone. Nothing about the
crash is specific to that property; `compile: true` on `set_module_input`,
`set_static_switch` or `add_module` reaches the identical finalizer. The window is the
compile itself, so it is timing-dependent on the tick landing inside it rather than
deterministic, which is worse: the same call sequence had already succeeded several
times in the same session before it fired.

## Evidence

`Saved/Logs/EAContentExamples58.log`, assert at `2026.08.27-14.41.59.225` UTC (19:41
local; logs are UTC+0, machine UTC+5), preceded by
`LogThreadingWindows: Error: Runnable thread Background Worker #3 crashed.` Callstack
innermost first:

```
FDebug::CheckVerifyFailedImpl2()                        AssertionMacros.cpp:797
VectorVM::Runtime::SetupBatchStatePtrs()                VectorVMRuntime.cpp:421
VectorVM::Runtime::ExecVectorVMState()                  VectorVMRuntime.cpp:2286
FNiagaraScriptExecutionContextBase::ExecuteInternal()   NiagaraScriptExecutionContext.cpp
FNiagaraScriptExecutionContextBase::Execute()
FNiagaraEmitterInstanceImpl::Tick'::<lambda_3>::operator()()
FNiagaraEmitterInstanceImpl::Tick()
FNiagaraSystemInstance::Tick_Concurrent()
FNiagaraSystemSimulation::FlushTickBatch()
FNiagaraSystemSimulation::AddSystemToTickBatch()
FNiagaraSystemSimulation::Tick_Concurrent()
TGraphTask<FNiagaraSystemSimulationTickConcurrentTask>::ExecuteTask()
LowLevelTasks::FScheduler::WorkerLoop()
```

The two immediately preceding compiles of the same asset are in the same log —
`LogNiagara: Compiling System NiagaraSystem /Game/Atlantis/VFX/NS_FishSchool.NS_FishSchool
took 1.079967 sec` at `14.37.12` and `0.124507 sec` at `14.37.27` — with the actor
`PWFISH_Probe` spawned and `IsActive()` verified `true` between them.

## Guilty source, and the fix is one line the codebase already has

`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraEditTypes.cpp:1567-1606`,
`FinalizeNiagaraEdit` — the shared finalizer every `niagara.*` edit verb routes
`compile`/`save` through:

```cpp
    bool FinalizeNiagaraEdit(const FNiagaraResolvedTarget& Target, const FNiagaraEditOptions& Options, bool& bOutCompiled, bool& bOutSaved)
    {
        ...
        if (Options.bCompile)
        {
            if (Target.System)
            {
                ...
                if (bHasSpawnSource && bHasUpdateSource)
                {
                    bOutCompiled = Target.System->RequestCompile(true);   // <-- live instances still ticking
                }
```

No `KillSystemInstances` anywhere in that function. `niagara.add_emitter` has the same
hole at `NiagaraHandler.cpp:96` (`bCompiled = System->RequestCompile(true);`).

Meanwhile the helper exists and is exported for exactly this
(`Handlers/Niagara/NiagaraInstanceUtils.cpp:12`, whose own header comment says it is the
"Replacement for `FNiagaraEditorUtilities::KillSystemInstances`" because the engine one
is not exported):

```cpp
    void KillSystemInstances(const UNiagaraSystem& System)
    {
        for (TObjectIterator<UNiagaraComponent> It; It; ++It)
        {
            UNiagaraComponent* Component = *It;
            if (Component && Component->GetAsset() == &System)
            {
                Component->DestroyInstance();
            }
        }
    }
```

and four handlers already call it before mutating — `NiagaraHandler.cpp:202`
(`remove_emitter`), `NiagaraEditHandler.cpp:1032`, `:2982`, `:3037`,
`NiagaraAdvancedEditHandler.cpp:179`, `NiagaraJsonHelpers.cpp:140`. So the invariant is
understood in this codebase; the shared compile path just does not honour it.

Fix: call `PinWrightNiagara::KillSystemInstances(*Target.System)` at the top of the
`Options.bCompile` / `Target.System` branch in `FinalizeNiagaraEdit`, and the same in
`niagara.add_emitter` before its `RequestCompile`. That is what
`FNiagaraSystemToolkit` does in the editor (it kills instances and rebuilds them from
`FNiagaraSystemUpdateContext`), which is why the same edit through the UI is safe.

A regression test can assert the invariant without needing the crash: spawn a component
on a fixture system, activate it, run an edit with `compile:true`, and assert the
component's instance was destroyed (`GetSystemInstanceController()` null / `IsActive()`
false) before the compile returned. Against current HEAD the instance is still live.

## Impact

Critical, and it lands on the **only** verification workflow the Niagara docs currently
permit. `render.capture_asset_preview` is already unusable
(`B-niagara-edit-with-open-asset-editor-slate-crash` and siblings), so the documented way
to see whether an authored system works is to spawn it into the level and capture. Doing
that means a live component exists for the whole authoring session — and then every
`compile: true`, which is the normal way to make an edit take effect, is a coin flip on
killing the editor for everyone in it. Three of this session's editor deaths were Niagara
authoring; this one is directly attributable.

## Workaround

Delete or deactivate every `ANiagaraActor` playing the system before any call carrying
`compile: true`, and re-spawn afterwards:

```
actor.delete {actorName: "PWFISH_Probe"}
// ... all edits with compile:true ...
niagara.spawn_actor / effect.activate_niagara
```

`effect.deactivate_niagara` is likely enough, but `DestroyInstance()` is what the helper
does and only `actor.delete` guarantees it from the client side.

## Distinct from related tickets

- `B-model-compile-live-niagara-mesh-renderer-raytracing-assert` is the mirror-image
  hazard on the **mesh** side — rebuilding a static mesh while a live Niagara mesh
  renderer draws it asserts in the ray-tracing gather. Same "live Niagara instance versus
  an asset being rebuilt under it" family, different asset, different assert, different
  fix site.
- `B-niagara-compile-wait-does-not-wait` is about `niagara.compile`'s `wait` semantics,
  not about what is running during the compile.
- `B-niagara-authored-emitter-forces-inert` is the `add_emitter`/`RebuildEmitterNodes`
  defect; `niagara.add_emitter` happens to share this missing-kill hole at `:96`, but the
  two faults are independent.

severity rationale: impact=`appError` on a worker thread killing the shared editor and every agent's unsaved work x reach=`compile: true` is the standard argument on every `niagara.*` edit verb, and the documented verification workflow requires a live component to be present the whole time -> Critical.

## History
- `#1-initial-repro` `OPEN` reporter — 2026-08-27, UE 5.8, Atlantis map build. Hit while authoring `/Game/Atlantis/VFX/NS_FishSchool` with `PWFISH_Probe` spawned and `IsActive()` confirmed `true` (that check is itself from `B-niagara-authored-emitter-forces-inert` #2). `niagara.set_property {propertyPath:"MeshIndexBinding.BindingSourceMode", compile:true, save:true}` never returned (`WinError 10054`); log shows `Runnable thread Background Worker #3 crashed` then `Assertion failed: DataSetIdx < ExecCtx->DataSets.Num() [VectorVMRuntime.cpp:421]` at `2026.08.27-14.41.59.225` UTC, callstack `SetupBatchStatePtrs` <- `ExecVectorVMState` <- `FNiagaraScriptExecutionContextBase::Execute` <- `FNiagaraEmitterInstanceImpl::Tick` <- `FNiagaraSystemInstance::Tick_Concurrent` <- `FNiagaraSystemSimulation::Tick_Concurrent` on a `LowLevelTasks` worker. Root cause read from source in this checkout: `FinalizeNiagaraEdit` (`NiagaraEditTypes.cpp:1567-1606`) calls `Target.System->RequestCompile(true)` with no preceding `KillSystemInstances`, and `niagara.add_emitter` repeats it at `NiagaraHandler.cpp:96`, while the helper `PinWrightNiagara::KillSystemInstances` (`NiagaraInstanceUtils.cpp:12`) is already called by six sibling call sites (`NiagaraHandler.cpp:202`, `NiagaraEditHandler.cpp:1032/2982/3037`, `NiagaraAdvancedEditHandler.cpp:179`, `NiagaraJsonHelpers.cpp:140`). Timing-dependent, not deterministic — several earlier `compile:true` calls on the same live system in the same session survived. Workaround used afterwards: `actor.delete` the preview actor before any `compile:true`, re-spawn to capture.
- `#2-quiesce-before-compile` `IN-REVIEW` developer — `NiagaraEdit::FinalizeNiagaraEdit` now calls `PinWrightNiagara::KillSystemInstances(*Target.System)` at the top of its `Options.bCompile`/`Target.System` branch, before the `RequestCompile(true)` (`Handlers/Niagara/NiagaraEditTypes.cpp`), which covers every `niagara.*` edit verb plus `niagara.compile` since they all route `compile`/`save` through that finalizer. `niagara.add_emitter` got the same call immediately before its own `RequestCompile` (`Handlers/Niagara/NiagaraHandler.cpp`); `remove_emitter` already quiesced before mutating and needed nothing. The emitter-target branch of the finalizer had the same hole with a wider blast radius — `UNiagaraSystem::RequestCompileForEmitter` recompiles every loaded system using the emitter — so a new sibling helper `PinWrightNiagara::KillSystemInstancesUsingEmitter(const UNiagaraEmitter&, const FGuid&)` (`Handlers/Niagara/NiagaraInstanceUtils.h/.cpp`) kills the instances of every system whose `UsesEmitter` matches, and runs before that call. Verified against engine source that the guard does not merely move the crash: `DestroyInstance()` -> `FNiagaraSystemInstance::Deactivate(bImmediate=true)` drains the in-flight concurrent tick via `WaitForConcurrentTickAndFinalize` before releasing, and the `FNiagaraSystemUpdateContext` at the tail of `RequestCompile` reinitializes auto-activate components while `UNiagaraComponent::Activate` parks on `HasOutstandingCompilationRequests()` until the compile lands, so the preview restarts against the new compiled data instead of racing it. New regression test `PinWright.niagara.CompileQuiesce.KillsLiveInstancesBeforeRecompile` (`Source/PinWright/Private/Tests/Niagara/TestNiagaraCompileQuiesce.cpp`) counts `UNiagaraComponent::OnSystemInstanceChanged()` broadcasts — `DestroyInstance()`'s unconditional last act — and asserts the target system's component is quiesced exactly once on `compile:true`, zero times on `compile:false`, and that a component bound to a different system is never touched; against pre-fix source all three counts are 0. Not compiled or run here (orchestrator owns builds).

- `#2-circumstantial-second-editor-death-no-dump` `OPEN` reporter — 2026-08-27, UE 5.8, Atlantis map,
  shared editor with three agents. **Circumstantial: no assert text and no crash dump were produced,
  so this is a timing match, not a proven repro.** Recorded because the shape matches #1 exactly and
  because the missing dump is itself a diagnostic fact worth knowing about this failure mode.

  Sequence from `Saved/Logs/EAContentExamples58-backup-2026.08.27-15.31.35.log` (UTC):

  ```
  15.30.28  McpSafeLevelSave: saved /Game/Maps/Atlantis
  15.30.35  LogNiagara: Compiling System NiagaraSystem /Game/Atlantis/VFX/NS_Bubbles_Stream took 0.938789 sec
  15.30.41  LogFileHelpers: Saving Package: /Game/Atlantis/VFX/NS_Bubbles_Stream
  15.31.16  actor.delete: Deleted actor 'PWBUB_P1'
  15.31.35  actor.delete: Deleted actor 'PWTEST_Bubbles'
  <log ends; process gone; port 27145 refuses; no UnrealEditor process; no new folder in Saved/Crashes>
  ```

  At the moment of that compile there were **three** live `UNiagaraComponent`s playing
  `NS_Bubbles_Stream` in the open level: the placed `VFX_BubbleVent_A` plus the two preview actors
  `PWBUB_P1` and `PWTEST_Bubbles` — and the two deletes that would have satisfied the ticket's own
  workaround happen 35 s and 54 s **after** the compile, not before it. That is the documented
  ordering hazard, executed in the wrong order by an agent that did know about the workaround.

  Two details that differ from #1 and are worth recording:

  - **The death produced no `Saved/Crashes` entry and no assert line in the log.** #1 left a full
    `Assertion failed: DataSetIdx < ExecCtx->DataSets.Num()` with a `LowLevelTasks` worker callstack.
    Here the log simply stops. An agent triaging the next occurrence should not conclude "no assert,
    therefore not this bug" — absence of a dump is compatible with the same worker-thread `appError`.
  - **Blast radius, again on a different agent.** The agent that issued the compile was repairing
    `NS_Bubbles_Stream`; the agent that lost work was placing VFX. Four `ANiagaraActor`s spawned
    between the last level save and the death were gone on restart, and of the surviving seven,
    **eight of thirteen `niagara.modify_parameter` User overrides had reverted to the asset default**
    (verified by decoding `NiagaraComponent0.OverrideParameters.ParameterData` off the reloaded
    components, not by trusting the setter echo). Re-applying and re-verifying them cost ~25 RPCs.
    This is the "two of five crashes killed the wrong agent" pattern the map spec's § Shared-editor
    safety rules describes, now at three of eight.

  Nothing new about the fix: `FinalizeNiagaraEdit` still needs `KillSystemInstances` before
  `RequestCompile`. What this encounter adds is that the client-side workaround is not sufficient in
  a shared editor, because **a placed level actor is also a live component** — an agent that deletes
  only its own `PWxxx_` preview actors still leaves the shipped `VFX_*` actor of that system ticking.
