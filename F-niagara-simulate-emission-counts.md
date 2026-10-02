---
id: F-niagara-simulate-emission-counts
title: "No verb proves a Niagara system emits: nothing reads a live particle count, so an inert system validates green"
status: DONE
severity: High
category: feature
tags: [niagara, validate, simulate, particle-count, emission, headless, gap-analysis-2026-09-30]
---

# No verb proves a Niagara system emits particles

`niagara.validate` is static, plus a survey of placed-component activation
(`Handlers/Niagara/NiagaraInspectHandler.cpp:804-892`, activation at `:551-615`). It never ticks
anything. No PinWright source calls `GetNumParticles`, uses SimCache, or calls
`FlushPendingTicks_GameThread`. So a system can compile clean, validate `valid: true`, and still
emit nothing: SpawnRate 0, a burst count of 0, a spawn module in the wrong stage, a
`Once` emitter that completes before a sample, or a force stack that never moves (see
`B-niagara-authored-emitter-forces-inert`). The only evidence available today is visual
(`effect.step_and_capture`), and it needs a placed actor.

Gap-analysis context (pinwright.com/compare row "Niagara emission validation", 2026-09-30):
no competitor reads a particle count. A grep of ue-mcp, ChiR24, VibeUE and Monolith for
`GetNumParticles|GetNumActiveParticles|ParticleCount` finds nothing.
- ue-mcp's "yes" is a static heuristic (`NiagaraHandlers.cpp:2660-2724`): enabled + a function
  call whose name contains "Spawn" in EmitterUpdate + an enabled renderer. It misses spawn
  modules in EmitterSpawn, rate 0 and burst 0.
- Monolith: the same `Contains("Spawn")` heuristic (`MonolithNiagaraActions.cpp` ~`:10343`), and
  a `preview_system` that renders pixels only.
- ChiR24: `advance_simulation` returns only the step count.
- VibeUE `DebugActivation` (`UNiagaraService.cpp:1068`): static.

A measured count would put PinWright ahead, not level. Do NOT add a name-based "no spawn module"
warning to `niagara.validate`: per `docs/rpc-design.md` §1 a warning must be derived from the
thing it warns about, not from something that correlates with it.

## Proposed verb

`niagara.simulate` is a read verb. It uses a transient world only and leaves every package
dirty flag as it found it.

**Params**
- `assetPath`: required, a Niagara System.
- `seconds`: required, no safe default, at most 60.
- `deltaTime`: default 1/30, at least 0.0001.
- `sampleEvery`: optional, in steps.
- `userParams`: optional `{name: value}` overrides, applied to the transient component only.

**Flow**
1. Gate: the compile has completed successfully (`niagara.compile_status` logic), and the
   data-interface check is consistent (`NiagaraDataInterfaceConsistency` /
   `NiagaraTickPreflight`). Otherwise refuse, because a mismatched data interface asserts in the
   VectorVM on its first tick.
2. Build a transient `FPreviewScene` (pattern at `Handlers/Render/MeshPreviewCaptureUtils.cpp:31`).
   Create a `NewObject<UNiagaraComponent>(GetTransientPackage(), RF_Transient)` and apply
   `SetAutoActivate(false)`, `SetAsset`, `SetAllowScalability(false)` (`NiagaraComponent.h:887`;
   EffectType culling otherwise deactivates it with no viewer) and `SetForceSolo(true)` (:307).
   Then `EnsureSystemInstance` (`Handlers/Render/CaptureSubjectProviders_Niagara.h:214`).
3. Advance with fixed steps via `AdvanceSimulation(k, dt)` (`NiagaraComponent.h:661`). That call is
   a silent no-op when there is no controller or `dt <= SMALL_NUMBER`, so verify the age moved,
   as `AdvanceToTime` (:456) already does. If any emitter is GPU, run
   `World->SendAllEndOfFrameUpdates()`, then
   `FNiagaraGpuComputeDispatchInterface::Get(World)->FlushPendingTicks_GameThread()`
   (`NiagaraGpuComputeDispatchInterface.h:169`), then `FlushRenderingCommands()`, before each
   sample. This is the same sequence SimCache uses (`NiagaraSimCache.cpp:709-724`). Without it,
   `FNiagaraEmitterInstance::GetNumParticles` (`NiagaraEmitterInstance.h:72`, impl `.cpp:34-55`)
   returns `TotalSpawnedParticles`, a cumulative guess that ignores deaths.
4. Sample `Controller->GetSystemInstance_Unsafe()->GetEmitters()`: per emitter,
   `GetNumParticles()` and `GetExecutionState()`, plus the system age and execution state.
5. Destroy the component and the scene.

**Response**
`{emitters:[{name, simTarget, samples:[{t, count, state}], maxCount, emitted, countExact}],
system:{achievedAgeSeconds, executionState, stallReason?, deterministic}}`.
- `emitted` is `maxCount > 0`, measured.
- `countExact` is false when a GPU fence had not passed.
- `stallReason` comes from `DescribeAdvanceStall` (`CaptureSubjectProviders_Niagara.h:330`).
  Per §17, a run that advanced nothing reports the stall; it never reports `emitted:false`.
- `deterministic` comes from `ReadDeterminism` (:738).

**Pitfalls to design for**
- Take the first sample after tick 1, not at t=0. Short bursts die between samples, which is why
  the response publishes `maxCount`.
- A system whose emitters are all Complete makes `AdvanceSimulation` return early
  (`NiagaraSystemInstance.cpp:993-996`). Report per-sample state so a 0 after completion is not
  read as "never emitted".

**Fallback** if the GPU fence path proves flaky: `CaptureNiagaraSimCacheImmediate(...,
bAdvanceSimulation=true)` (`NiagaraSimCacheFunctionLibrary.h:82`) per frame, then
`UNiagaraSimCache::GetEmitterNumInstances` (`NiagaraSimCache.h:512`). It handles the GPU readback
internally but is heavier because it copies attributes.

**Engine range:** every API above exists on 5.3-5.8. `GetEmitters()` changed its return type in
5.4 (5.3 returns `TArray<TSharedRef<...>>&`, 5.4+ a `TArrayView`); range-for works on both.

`niagara.validate`'s wiki should point here for "does it emit?".

Related, not duplicates: `F-effect-step-and-capture-atomic` (placed actor, pixels; its note asks
the `effect.*` verbs to report a particle count), `B-niagara-validate-green-while-component-inactive`,
`B-effect-advance-unbounded`.

## Acceptance
- A CPU burst system reports `maxCount > 0` and `emitted: true`.
- An emitter whose SpawnRate is 0 reports `emitted: false`, with no `stallReason`, and the
  system's age advanced.
- A GPU emitter reports `countExact: true` after the flush, with a count within the CPU twin's
  range.
- A system with mismatched data interfaces is refused before any tick.
- No actor is left in the editor world, and every package dirty flag is unchanged.
- A system that cannot advance reports `stallReason` and no `emitted` verdict.

Effort M. Risk: GPU fence timing (SimCache fallback) and crash-on-tick (gated by the existing
preflight).

## History
- `#1-filed-from-gap-analysis` `OPEN` reporter — Filed from the 2026-09-30 competitor gap analysis (compare row "Niagara emission validation": PinWright partial; ue-mcp "yes" is a static name heuristic; nobody reads live counts). Engine citations are UE 5.8 source reads; PinWright citations are current plugin source. Nothing was built or run.
- `#2-verb-implemented` `IN-REVIEW` developer — Added `niagara.simulate` in new `Source/PinWright/Private/Handlers/Niagara/NiagaraSimulateHandler.cpp`. Flow: `RequireRenderer` first (Niagara will not instance anything under -NullRHI since `UNiagaraSystem::IsReadyToRunInternal` needs `FApp::CanEverRender()`, so every count would be a structural zero; the verb answers `RENDERING_UNAVAILABLE`); validate `seconds` (required, (0,60]), `deltaTime` (default 1/30, >= 0.0001), at most 3600 steps (`INVALID_ARGUMENT`), optional `sampleEvery`; compile gate copied from effect.spawn_niagara (`WaitForSystemCompile` + `IsSystemReadyAfterCompileWait`, `SYSTEM_NOT_COMPILED`); `CheckDataInterfaceCounts` Mismatched -> `NIAGARA_DATA_INTERFACE_MISMATCH` before any tick; transient `UNiagaraComponent` (`SetAutoActivate(false)`, `SetAllowScalability(false)`) in a private `FPreviewScene`, instance brought up with the existing `EnsureSystemInstance` (force-solo); one `AdvanceSimulation(1, dt)` per step, then for GPU systems `SendAllEndOfFrameUpdates` + `FlushPendingTicks_GameThread` + `FlushRenderingCommands`; every emitter read after EVERY step (`GetNumParticles`, `GetExecutionState`, GPU fence `ParticleCountReadFence <= ParticleCountWriteFence` for exactness); `maxCount` is over exact readings only and `sampleEvery` only thins the published samples. Verdicts: per emitter `emitted`/`maxCount`, or `notMeasuredReason` (stall, no GPU dispatch interface, fence never passed, or Disabled at every step) — never a fake 0; top-level `emitted` only when the age advanced and either something emitted or every emitter was measured. `stallReason` from `DescribeAdvanceStall` when the age did not move. Teardown: DeactivateImmediate, RemoveComponent, DestroyComponent, scene destroyed with `r.ForceGCOnPreviewSceneExit` suppressed; `FScopedPackageDirtyRestore` over the system and its emitters. Added `niagara.simulate` to the tick-unsafe table (`Dispatch/SafePoint.cpp`) and to `PWRenderGuardedVerbs` (`Tests/Infra/TestRenderingUnavailableGuard.cpp`). NOT implemented: `userParams` (optional in the proposal; deferred until someone needs it). Wiki: `### niagara.simulate` in `docs/wiki-src/niagara.md`, plus a pointer at the top of `### niagara.validate`. CHANGELOG entry. Tests (`Tests/Niagara/TestNiagaraSimulate.cpp`, filter `PinWright.niagara.simulate`): `SpawningEmitterReportsParticles` (stock SimpleExplosion: emitted true, maxCount > 0, age advanced, package dirty flag, editor-level actor count and live instance count unchanged), `ZeroSpawnCountReportsNotEmitted` (duplicate with every `SpawnBurst_Instantaneous.Spawn Count` rapid-iteration constant set to 0: emitted false, every maxCount 0, no stallReason, age advanced), `GpuEmitterCountIsReadOrExplained` (stock AttributeReaderTrails: every GPU emitter carries exactly one of emitted / notMeasuredReason; skip marker when none was readable), `RefusesUnboundedRuns`; plus `PinWright.infra.rendering_guard.EveryGuardedVerbRefusesWithoutRenderer` now covers the verb. All four skip with the marker under NullRHI. Not covered by a test: the data-interface-mismatch refusal (no way to build a Mismatched fixture, same limitation as the DI consistency tests). Only compile-checked on UE 5.8; the 5.3-5.7 engine range was not built.
- `#3-acceptance-gaps-closed` `IN-REVIEW` developer — Closed the acceptance bullets `#2` left untested, plus one verb defect found while doing it. **Verb fix** (`NiagaraSimulateHandler.cpp`): the compile gate excludes GPU shaders, and a GPU emitter whose shader map is incomplete is skipped by the dispatcher (`IsShaderMapComplete_RenderThread`) while its count fence still passes, so it read an exact-looking `emitted:false` / `maxCount:0`. Now, when `HasOutstandingCompilationRequests(true)` holds at run start, each GPU emitter gets `notMeasuredReason` instead (wiki updated). **New tests** in `Tests/Niagara/TestNiagaraSimulate.cpp`. (1) `PinWright.niagara.simulate.GpuCountMatchesCpuTwin`: one duplicate of SimpleExplosion; its `SimpleSpriteBurst` emitter is simulated on CPU, then flipped to `GPUComputeSim` the way the editor does it (`SimTarget` + `PostEditChangeVersionedProperty`, which requests the emitter recompile), recompiled (VM through the bounded `WaitForSystemCompile`, GPU shaders through `UNiagaraSystem::WaitForCompilationComplete(true)`, because `FNiagaraShaderScript::FinishCompilation` is not exported) and simulated again with the same seconds/deltaTime. It asserts GPU `countExact:true`. Tolerance: a single `SpawnBurst_Instantaneous` at t=0 with Spawn Probability 1 gives a peak equal to the authored `Spawn Count`, independent of the seed, so it asserts CPU maxCount == Spawn Count (premise) and GPU maxCount == CPU maxCount. With probability < 1 the only honest bound is 1 <= GPU max <= Spawn Count. It skips only under NullRHI, or when the GPU count is unreadable (the reason is quoted). (2) `PinWright.niagara.simulate.RefusesDataInterfaceMismatchBeforeAnyTick`: a REAL fixture, no hook. A compiled duplicate (premise: Consistent, nothing running it) gets one extra default entry in one script's `ResolvedDataInterfaces`, reached reflectively the same way `NiagaraDataInterfaceConsistency.cpp` does, which re-measures Mismatched. The test asserts `NIAGARA_DATA_INTERFACE_MISMATCH`, zero live instances after the call (nothing was instanced, so nothing ticked), and that the corruption is still in place. The entry is popped before teardown. Caveat: if the gate regresses, this test crashes the suite host in the VectorVM instead of going red. (3) `PinWright.niagara.simulate.StalledRunReportsReasonAndNoVerdict`: `fx.Niagara.SystemSimulation.SkipTickDeltaSeconds=1` (saved and restored) freezes the age at 0. Asserts `stallReason` names the cvar, there is no top-level `emitted`, and every emitter has `notMeasuredReason` and no `emitted` / `maxCount`. (4) `PinWright.niagara.simulate.DirtyPackageStaysDirty`: the dirty-start half of §11 (`SpawningEmitterReportsParticles` already covers clean-stays-clean, the editor-level actor count and the live instance count). On the 'SpawnRate is 0' bullet: covered by `ZeroSpawnCountReportsNotEmitted` through a burst count of 0. The verb does not care which spawn source produced zero particles, and SimpleExplosion has no SpawnRate module; authoring one would add a module insert and compile while testing nothing more about the verb. Single-file compiles of both files succeed on UE 5.8; `check_test_ids` and `check_test_skips` report nothing. Not run: the coordinator holds the editor/test slot.
- `#4-verified-linux` `DONE` tester — All 8 `PinWright.niagara.simulate.*` tests pass with no skip markers in the round-4 scoped run `4d719404` (offscreen, Linux Vulkan, UE 5.8; 1369/1370, the one failure is an unrelated layout test). The first four also passed in the round-1, round-2 and final full suites. Acceptance: CPU burst `emitted:true` with `maxCount > 0` (`SpawningEmitterReportsParticles`); zero spawn `emitted:false`, no `stallReason`, age advanced (`ZeroSpawnCountReportsNotEmitted`, burst Spawn Count rather than SpawnRate since the fixture has no SpawnRate module); GPU `countExact:true` equal to its CPU twin (`GpuCountMatchesCpuTwin`, measured, no skip); mismatched data interface refused before any tick (`RefusesDataInterfaceMismatchBeforeAnyTick`, real corrupted fixture); no leftover actor or instance and dirty flags unchanged both ways (`SpawningEmitterReportsParticles`, `DirtyPackageStaysDirty`); a stalled system reports `stallReason` and no verdict (`StalledRunReportsReasonAndNoVerdict`). PinWright commit `29a94e91`.
- `#5-duplicate-not-compiled` `IN-REVIEW` developer — `DirtyPackageStaysDirty` and `GpuCountMatchesCpuTwin` failed the waves 2+3 full run with `LogNiagara: Error initializing data interfaces. Completing system.` **Root cause: test order and fixture, not the NiagaraEditTypes.cpp wave** (that diff only touches standalone-script assets). `effect.step_and_capture.ShortBurstAtEarlySampleIsVisible` compiled the stock SimpleExplosion at 22:07:22. Every later in-memory duplicate then reported up to date, so the verb's compile gate let it run with no compile of its own (no `Compiling System` line before the error). It then failed binding the system scripts' data-interface function tables (`NiagaraSystemInstance.cpp:1775`, `Complete(true)`). In the passing run the source still carried its on-load deferred compile request, which the duplicate inherited and the gate compiled. The tests that already forced a compile (`ZeroSpawnCount…`, `RefusesDataInterfaceMismatch…`) were unaffected. **Fixes:** (1) The tests now force-compile their duplicate with a shared `CompileDuplicate` helper (`RequestCompile(true)` + bounded `WaitForSystemCompile`). (2) The 'empty code/message' was not a verb defect. The CPU run SUCCEEDED and reported a stall, with no `emitted` and every emitter `notMeasuredReason`, as §17 requires. The test printed the empty error fields of a success. Assertion messages now print success, code, message and `stallReason` through `DescribeRun`. (3) Verb: the stall was honest but generic ('Complete'). It now detects an instance that is already complete before the first step and names the cause: data-interface init failure, and that an in-memory duplicate not compiled since has been seen to do this; run niagara.compile. Wiki row updated. Why not auto-compile or refuse up front: every pre-instance check the engine offers (IsReadyToRun, script compile status, compile queue, the compiled/resolved DI count check) passes on such a duplicate, so it cannot be told apart before instancing. Forcing a recompile on every call would make a read verb rewrite compiled data on every system. The ticket's contract for a run that cannot advance is a stallReason with no verdict, and the reason now carries the remedy. Single-file compiles of `NiagaraSimulateHandler.cpp` and `TestNiagaraSimulate.cpp` succeed; check_test_ids and check_test_skips report nothing.
