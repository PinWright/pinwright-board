---
id: F-niagara-simulate-emission-counts
title: "No verb proves a Niagara system emits: nothing reads a live particle count, so an inert system validates green"
status: OPEN
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
