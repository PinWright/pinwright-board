---
id: B-call-function-float-arg-loses-decimal
title: "object.call_function silently mis-marshals some fractional float arguments: 0.18 arrives as a value that clamps to 1.0, while 0.4 and 0.2 arrive correctly"
status: OPEN
severity: High
category: bug
tags: [object, call_function, rpc, marshalling, float, silent-wrong, blueprint, args]
---

# `object.call_function` passes a wrong float for some fractional values, with no error

Calling a Blueprint function with `args: {"Value": 0.18}` runs the function but the parameter that
arrives is **not 0.18**. The call returns a clean `{"void": true}`; nothing indicates a bad argument.

## Repro (measured 2026-09-03T03:1x Z, EAContentExamples58, port 27145)

Target `/Game/FPS/UI/Test/BP_HUDTestPawn`, function `DebugHoldHealth(float Value)`. Its whole body,
from `blueprint.decompile_function`, is:

```
set bHoldHealth = true
set bRegen = false
%n0 = call FClamp(Value: $Value, Min: 0.0, Max: 1.0)
set Health01 = %n0
call_dispatcher OnHealthChanged(Health01: %n0)
```

So `Health01` is exactly `clamp(Value, 0, 1)` and is directly readable with `property.get`:

| `args` sent | `Health01` read back | Verdict |
|---|---|---|
| `{"Value": 1}`    | `1`   | ok |
| `{"Value": 0.18}` | **`1`** | **wrong — three separate attempts** |
| `{"Value": 0.4}`  | `0.4` | ok |
| `{"Value": 0.2}`  | `0.2` | ok |

`bHoldHealth` read back `true` after the 0.18 call, proving the function *ran* — only the argument
was wrong. A `property.set` of `Health01 = 0.18` in the same session also reported
`applied: true, value: 0.18000000715255737` and read back `1` on the next call, so the 0.18 path is
wrong on more than one verb.

## Why the value clamps to 1

`FClamp(x, 0, 1)` returns 1 for any `x >= 1`. An arriving `18` — i.e. **`0.18` with its decimal
separator lost** — clamps to exactly 1, which is what is observed. `0.4` and `0.2` have a single
fraction digit; `0.18` has two. That points at a string round-trip in the argument path that is
locale- or format-sensitive (this machine is a comma-decimal locale: PowerShell prints `138,2` for
138.2). It is a guess about the mechanism, not about the observation.

## Why this is High

It is silent and it corrupts *measurements*, not just state. This argument was the hold value for a
HUD vignette test, so every frame captured with it was labelled 0.18 while the HUD was actually
rendering full health — a reviewer reads that as "the vignette is broken at low health" and files a
defect against innocent code. Two consecutive review rounds on this project already burned a
world-lock slot each chasing HUD state that turned out to be a test harness lying about its own
input. Any caller passing a two-decimal float to a Blueprint function is exposed.

## Workaround

Pass values with a single fraction digit where the test allows it (`0.2` instead of `0.18`), and
**always read the driven property back after the call** rather than trusting the argument landed.
A read-back bracketing every capture is the only reliable protocol.

## Fix

Marshal numeric arguments through the JSON numeric type end to end; if any stage stringifies, use an
invariant-culture round trip. Fail loudly on a value that cannot be converted rather than passing a
silently different number. A regression test should cover multi-digit fractions (0.18, 0.075, 1.25)
against a float parameter and assert the received value, not just that the call succeeded.

## History
- `#1-filed` `OPEN` reporter — Found while verifying a HUD low-health vignette. `DebugHoldHealth(0.18)`
  left `Health01` at `1` on three separate attempts in one PIE session, while `0.4` and `0.2` landed
  exactly; the callee is a five-node function whose only behaviour is `clamp(Value,0,1)`, so the
  argument is the only variable. Confirmed the function executed by reading back `bHoldHealth: true`.
