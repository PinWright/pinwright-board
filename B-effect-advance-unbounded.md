---
id: B-effect-advance-unbounded
title: "effect.advance_simulation accepts an unbounded tick count and can freeze the game thread"
status: OPEN
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

## History
- `#1-source-scan` `OPEN` reporter -- Source-confirmed against the handler and UE 5.8 loop; no
  editor call was made under the scan constraints.
