---
id: F-niagara-audit-level-data-interfaces
title: "No verb answers 'is anything loaded here fatal on its next tick' — the Niagara data-interface check is asset-scoped only, so a capture session has no level-wide pre-flight"
status: OPEN
severity: Medium
category: feature
tags: [niagara, data-interface, audit, level-scope, capture, preventative-check, vectorvm, editor-kill]
encounters: 1
lastSeen: 2026-08-28
---

# Nothing sweeps the level for the state that kills the editor

`B-niagara-di-count-mismatch-vectorvm-assert-kills-editor` established the fault: a Niagara system
whose compiled `DataInterfaceInfo` count disagrees with its resolved set asserts in `VectorVM`
(`DataSetIdx < ExecCtx->DataSets.Num()`) on a concurrent worker the next time anything ticks it.
That is an `appError`, so it takes the whole shared editor down, minutes after and unrelated to the
write that armed it.

Every surface that can see that state today needs a `systemPath` the caller already suspects.
Nothing walks what is loaded and answers the question a capture session actually has: *is anything
here fatal on its next tick.*

## Why a pre-flight on `sequencer.set_playhead` is the wrong shape

`E-niagara-validate-no-data-interface-check` `#2` refused that shape. Verified, and the engine
evidence makes the case stronger than the argument given:

- **`set_playhead` is per-frame hot.** Its own registration calls it "the deterministic driver for
  frame bursts (set position -> capture -> repeat)"
  (`Handlers/Sequencer/SequenceHandler.cpp:1976-1982`), and `camera.animation_shots` plus both level
  capture-subject providers step the same `SequencePlayheadUtils::ApplyPlayheadPosition` per frame
  (`Handlers/Render/AnimationShotsHandler.cpp:705`,
  `Handlers/Render/CaptureSubjectProviders_Level.cpp:324,476`). A scan there runs on every frame of
  every burst.
- **The re-tick claim holds, and what ticks is the whole world — never the bindings.** Two
  independent paths, neither of them binding-aware. Paths below are under `C:\UE_5.8\Engine\`.
  - *The batched path, and the one that matters.* `FNiagaraWorldManager::Init` registers a
    `FNiagaraWorldManagerTickFunction` on the persistent level of **every** world including the
    editor world (`Plugins\FX\Niagara\Source\Niagara\Private\NiagaraWorldManager.cpp:463`, world
    hookup `:1143-1154`); `ExecuteTick` ignores `TickType` entirely (`:379-383`); and
    `FNiagaraWorldManager::ExecuteSimulations` ticks **every** `FNiagaraSystemSimulation` registered
    in that world (`:1799-1823`). Placed systems reach it by auto-activation:
    `UActorComponent::OnRegister` calls `Activate(true)` when the world is not a game world
    (`Source\Runtime\Engine\Private\Components\ActorComponent.cpp:1606-1613`) and
    `UNiagaraComponent` sets `bAutoActivate = true` (`NiagaraComponent.cpp:685`).
  - *The component path.* `UNiagaraComponent` sets `bTickInEditor = true`
    (`NiagaraComponent.cpp:684`) and `FActorComponentTickFunction::ExecuteTickHelper` runs such a
    component under `LEVELTICK_ViewportsOnly`
    (`Source\Runtime\Engine\Classes\GameFramework\Actor.h:4887`).
  - *The switch for both is realtime.* `UEditorEngine::Tick` picks `LEVELTICK_ViewportsOnly` over
    `LEVELTICK_TimeOnly` from `IsRealtime` (`Source\Editor\UnrealEd\Private\EditorEngine.cpp:1958`,
    world tick `:1968`), and `LEVELTICK_TimeOnly` runs no tick groups at all
    (`Source\Runtime\Engine\Private\LevelTick.cpp:1650`). Opening a Sequencer forces
    `AddRealtimeOverride(true, "Sequencer")` on every perspective level viewport
    (`Source\Editor\Sequencer\Private\LevelEditorSequencerIntegration.cpp:1248`, from `AddSequencer`
    at `:1486`), which **overrides** the user's own realtime toggle
    (`Source\Editor\UnrealEd\Public\EditorViewportClient.h:414-417`). So opening the sequence can
    start the whole level ticking. Sequencer itself never calls `UWorld::Tick` — there is no such
    call anywhere in Sequencer, MovieScene or LevelSequence — so the promotion *is* the mechanism.

  With a realtime viewport already on, the level was ticking before the sequence opened at all. The
  parent ticket's `#4` occurrence A reached the same fault with no sequencer in the picture (an
  autosave presave with three live components, 14-minute game-thread wedge).
- **A binding-scoped pre-flight would inspect the *safest* subset.**
  `FNiagaraSystemUpdateDesiredAgeExecutionToken::Execute` touches only
  `Player.FindBoundObjects(Operand)` and calls `SetForceSolo(true)` on them
  (`Plugins\FX\Niagara\Source\Niagara\Private\MovieScene\MovieSceneNiagaraSystemTrackTemplate.cpp:127,142`),
  which pulls a bound system **out** of the batched world simulation and onto its own component
  tick — and `UNiagaraComponent::TickComponent` early-returns when the component is not solo
  (`NiagaraComponent.cpp:936-940`). The systems such a gate would check are precisely the ones
  removed from the path that ticks everything else. It is unsound, not merely narrow. And even a
  whole-level scan bolted onto `set_playhead` would give false safety, because in a shared editor
  the arming write can land between the scan and the next frame.

The right shape is a read-only sweep the caller runs **once**, deliberately — before a capture
session, or after a `niagara.*` write — not a gate on a hot verb.

## What already exists

- `PinWrightNiagara::CheckDataInterfaceCounts` / `FindOrphanResolvedDataInterfaces`
  (`Handlers/Niagara/NiagaraDataInterfaceConsistency.h`) — the measurement itself, per
  `UNiagaraSystem`, with the three-way `Consistent` / `Mismatched` / `Unverified` verdict. Complete;
  this ticket needs no new measurement code.
- `niagara.validate` now runs it (`Handlers/Niagara/NiagaraInspectHandler.cpp:275,467`, IN-REVIEW
  under `E-niagara-validate-no-data-interface-check`); `niagara.add_emitter` / `remove_emitter` gate
  their save on it; `niagara.list_orphan_data_interfaces` / `niagara.remove_orphan_data_interfaces`
  (`Handlers/Niagara/NiagaraAdvancedEditHandler.cpp:660`) name and repair the cause. **All of them
  take a `systemPath`.**
- `PinWrightNiagara::KillSystemInstances` (`Handlers/Niagara/NiagaraInstanceUtils.cpp:27`) already
  performs the walk this verb needs: `TObjectIterator<UNiagaraComponent>` filtered on `GetAsset()`.
- The audit contract is shared and already has four members — `level.audit`,
  `geometry.audit_static_meshes`, `landscape.audit_shape`, `skeleton.audit_skin_weights` — all
  deriving `pass` through `PinWrightAudit::FVerdict::DerivePass` (`Audit/AuditFramework.h`,
  `Docs/rpc-design.md` §18).

**This is a new verb, not a new check on `level.audit`.** That verb's subject is an actor and its
walk is `TActorIterator<AActor>` over one world (`Handlers/Level/LevelAuditUtils.cpp:828`); the
subject here is a `UNiagaraSystem` asset reached through live components, N actors share one system,
and a component in an asset-editor preview scene has no actor at all. What is reused is the audit
**contract**, not the audit verb.

## Proposed: `niagara.audit_level`

Read-only. Loads nothing, compiles nothing, saves nothing, dirties nothing.

- **Walks** live `UNiagaraComponent`s via `TObjectIterator`, deduplicated to the distinct
  `UNiagaraSystem` assets they hold, so `CheckDataInterfaceCounts` runs once per system rather than
  once per component. Default scope the editor world; `scope:"all"` includes preview-scene
  components, which tick too.
- **Reports per system**: `systemPath`; the `consistent` / `mismatched` / `unverified` verdict; the
  offending scripts with both counts on a mismatch; the orphan entries on request
  (`detail:"orphans"`, reusing `FindOrphanResolvedDataInterfaces`); and the components and owning
  actors that hold it — the caller needs the actor to know what to detach or delete.
- **Verdict** through `PinWrightAudit::FVerdict::DerivePass`. `mismatched` is an Error finding.
  `unverified` is an **unrunnable** row, never a clean one: a system nothing has compiled this
  session is unmeasured, and the framework exists precisely so unmeasured cannot read as a pass.
- **Capture workflow**: one call before the burst. On `pass:false`, pipe each named `systemPath`
  into `niagara.list_orphan_data_interfaces` / `niagara.remove_orphan_data_interfaces`, or detach
  the named components, then start the burst. One call per session instead of a scan per frame.

**Workaround:** enumerate the level's Niagara systems by hand (`level.get_actors`, or the
`asset-dumps/` mirror) and call `niagara.validate` per asset. Correct, but O(systems) calls and easy
to leave incomplete — which is why this is Medium and not the Critical of the crash it prevents: the
measurement and the remedy both already exist, only level-scope discovery is missing.

**Fix:** new `Handlers/Niagara/NiagaraAuditHandler.cpp` registering `niagara.audit_level`, built on
`Audit/AuditFramework.h` and calling the existing `CheckDataInterfaceCounts` /
`FindOrphanResolvedDataInterfaces`. One check in the table to begin with
(`data_interface_counts`); the table shape is what lets the next level-scope Niagara health check
(live component against stale compile state,
`B-niagara-compile-while-live-component-vectorvm-assert`) land as a row rather than a second verb.

## Addendum: scrubbing forces a world tick by explicit engine design

Later research found a second, stronger mechanism than the realtime-override one recorded above,
and it makes the binding-scoped pre-flight unsound rather than merely incomplete.

`Editor/Sequencer/Private/LevelEditorSequencerIntegration.cpp` registers `OnSequencerEvaluated`
on `OnGlobalTimeChanged`, so it fires on **every** playhead move, scrub and evaluation. After an
early-out for PIE/Simulate it calls `ReRenderLevelViewports()`, whose engine-authored comment
states the intent outright: *"Request a single real-time frame to be rendered to ensure that we
tick the world and update the viewport."* It calls `RequestRealTimeFrames(1)` on every level
viewport that is not already realtime; that sets `RealTimeUntilFrameNumber`, which makes
`IsRealtime()` true for the next frame, which makes `UEditorEngine::Tick` select
`LEVELTICK_ViewportsOnly` over `LEVELTICK_TimeOnly` (the latter runs no tick groups at all),
which runs a full tick-group pass, which reaches `FNiagaraWorldManagerTickFunction::ExecuteTick`
and then `ExecuteSimulations` — iterating **every** system simulation in the world with no
reference to Sequencer bindings.

The sibling call site comment (*"If realtime is off, this needs to be called to update the pivot
location when scrubbing"*) confirms the path exists specifically for the realtime-off case.
`RequestRealTimeFrames` has no overrides anywhere in `Engine/Source`.

Consequence: a corrupt system sitting in the level but **not bound by the sequence** is ticked and
crashes on any scrub. The pre-flight must enumerate every `UNiagaraComponent` in the editor world;
`Sequencer->FindBoundObjects(...)` is the wrong set. This is independent of, and additional to,
the persistent `AddRealtimeOverride(true, "Sequencer")` taken when the sequence is opened.

## History
- `#1-no-level-scope-preflight` `OPEN` reporter — Filed from the refusal recorded in
  `E-niagara-validate-no-data-interface-check` `#2`, after verifying its two load-bearing claims.
  `set_playhead` is per-frame hot (its own docstring plus three burst call sites). And the re-tick
  claim holds, traced end to end in engine source: Sequencer's realtime override -> the editor
  world's `LEVELTICK_ViewportsOnly` tick -> `ExecuteTickHelper`'s `bTickInEditor` branch ->
  `UNiagaraComponent`'s constructor setting that flag. What ticks is the whole level, never the
  bindings, which makes a binding-scoped pre-flight unsound rather than merely narrow — the refusal
  was right for a stronger reason than the one it gave. Not implemented; no plugin source touched.
- `#2-scrub-forces-world-tick-by-design` `OPEN` reporter — "Engine research strengthens the level-scope argument: OnSequencerEvaluated fires on every playhead move and calls ReRenderLevelViewports -> RequestRealTimeFrames(1), whose engine comment says it exists 'to ensure that we tick the world'. That promotes the next frame to LEVELTICK_ViewportsOnly, running the full tick-group pass and ExecuteSimulations over every system in the world. A binding-scoped pre-flight is unsound, not merely narrow — the set that crashes is every UNiagaraComponent in the editor world, not the sequence's bindings."
