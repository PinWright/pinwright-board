---
id: B-effect-advance-unbounded
title: "effect.advance_simulation accepts an unbounded tick count and can freeze the game thread"
status: IN-REVIEW
severity: Critical
category: bug
tags: [effect, niagara, advance-simulation, freeze, validation]
---

# Caller-controlled ticks run synchronously on the game thread

`effect.advance_simulation` reads `steps` and `deltaTime` directly from JSON, performs no finite,
positive, or upper-bound checks, and calls `UNiagaraComponent::AdvanceSimulation` before returning
success (`EffectHandler.cpp:846-887`).

UE 5.8 forwards positive values to `FNiagaraSystemInstance::AdvanceSimulation`, which first waits
for concurrent work and then performs `for (TickIdx = 0; TickIdx < TickCountToSimulate; ++TickIdx)`.
Its own comment says every `ManualTick` is forced onto the game thread
(`NiagaraSystemInstance.cpp:987-1007`). A caller can therefore submit millions or billions of
steps and monopolize the shared editor. The sibling `effect.step_and_capture` already has the fix
shape: finite/positive validation, `steps <= 10000`, and total simulated time `<= 60` seconds.

Negative/zero steps and `deltaTime <= SMALL_NUMBER` are the other side of the same missing
validation: the engine does nothing, while the handler still returns `success:true` and echoes the
requested step count.

## What it should do

Apply the same limits and finite checks as `step_and_capture`, returning `INVALID_PARAMS` before
calling the engine. Read back or report the actual simulated duration rather than only echoing
`steps`.

## Workaround

Use `effect.step_and_capture` for bounded stepping, or keep `steps` and `deltaTime` small and
positive.

## Related

- `F-effect-step-and-capture-atomic`

## Fix

Verdict: TRUE. Root cause was that `effect.advance_simulation` parsed caller-controlled `steps` and `deltaTime` and called Niagara directly, while the existing `step_and_capture` bounds and float-delta preparation stayed local to that handler. The shared preparation helper now rejects non-finite, non-positive, fractional, over-count, sub-minimum, and over-duration inputs with `INVALID_PARAMS`, while the runtime measures Niagara system age around the single `AdvanceSimulation` call. Both verbs report `requestedSeconds` separately from measured `simulatedSeconds`; a material shortfall adds `stoppedEarly: true` and an observable `stopReason`. The dispatcher defers the synchronous Niagara tick verb to a safe point.

Files changed: `Source/PinWright/Private/Handlers/VFX/EffectHandler.cpp`; `Source/PinWright/Private/Handlers/VFX/EffectRuntimeUtils.h`; `Source/PinWright/Private/Handlers/VFX/EffectRuntimeUtils.cpp`; `Source/PinWright/Private/Dispatch/SafePoint.cpp`; `Source/PinWright/Private/Tests/Assets/TestVFXHandlers.cpp`; `Source/PinWright/Private/Tests/Niagara/TestEffectStepAndCapture.cpp`; `Docs/wiki-src/effect.md`.

Test ids: `PinWright.effect.advance_simulation.RejectsOutOfRangeParams` (structural/unit coverage for handler validation, measured-age source shape, and the safe-point table; not run under the ticket's no-test scope); `PinWright.effect.step_and_capture.ContractAndStageOrder` (updated source/runtime contract coverage, not run); `PinWright.core.safe_point.TickUnsafeMethodsAreRegistered` (existing shared table-registration coverage, not run).

Deliberately unchanged: `EffectRuntimeUtils::Advance` still makes one Niagara `AdvanceSimulation` call and does not wrap or replay ticks; no Unreal Engine files, outer project files, generated/compiled skills, live editor state, or processes were changed. The new VFX rejection cases call the handler directly by design and do not prove dispatcher deferral; runtime/editor verification remains pending.

## History
- `#1-source-scan` `OPEN` reporter -- Source-confirmed against the handler and UE 5.8 loop; no
  editor call was made under the scan constraints.
- `#2-fix-bounded-advance` `IN-REVIEW` developer -- Applied the shared simulation bounds helper,
  safe-point table entry, actual-duration response, structural VFX coverage, and effect wiki update.
- `#3-measure-actual-age` `IN-REVIEW` developer -- Verified Niagara's void/manual-tick early exits
  and measured controller age around the single call; both effect verbs now separate requested
  from actual duration and expose conservative early-stop categories.
