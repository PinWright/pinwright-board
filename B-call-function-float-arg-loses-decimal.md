---
id: B-call-function-float-arg-loses-decimal
title: "object.call_function silently mis-marshals some fractional float arguments: 0.18 arrives as a value that clamps to 1.0, while 0.4 and 0.2 arrive correctly"
status: IN-REVIEW
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

## Fix (implemented)

**The reported root cause is refuted; a different, real silent-precision defect found in the same
audit is fixed.**

### Why the filed mechanism cannot happen

There is no string round trip and no locale-sensitive parse anywhere between the JSON number and a
`float` / `double` UFUNCTION parameter:

- The request body is parsed once, into a shared `FJsonObject` (`Transport/McpRequestCore.cpp:552-556`),
  and the `args` object is handed to the dispatcher **by shared pointer**
  (`McpRequestCore.cpp:788-798`) — nothing re-serializes it.
- `object.call_function` passes that `FJsonValue` straight to `ApplyJsonValueToProperty`
  (`Handlers/Reflection/ObjectCallFunctionHandler.cpp:250`), the same helper `property.set` uses
  (`Handlers/Utility/UtilityPropertyHandler.cpp:1339`).
- The scalar float / double branches take `ValueField->AsNumber()` (a `double`) and assign it
  (`Utils/PropertyImport.cpp:381-412`). `FCString::Atod` is only reached when the JSON value is a
  **string**, which is not what was sent.
- A "decimal separator lost" token cannot arrive silently: UE's JSON reader validates the number
  token against the JSON grammar FSA before converting, and state 2 (leading `0`) accepts only `.`,
  `e` or `E` — `018` is rejected as `Poorly formed Json Number Token`
  (`Engine/Source/Runtime/Json/Public/Serialization/JsonReader.h:790-793, 856`). The call would have
  failed at parse, not returned `{"void": true}`.

### The ticket's own evidence contradicts the ticket

`property.set Health01 = 0.18` reported `value: 0.18000000715255737`. That field is **not an echo of
the input** — it is a re-read of the live property after the write
(`UtilityPropertyHandler.cpp:1376-1379`), and `0.18000000715255737` is exactly `(double)(float)0.18`.
So on that machine, in that session, `0.18` marshalled into an `FFloatProperty` **exactly**. The
value reading `1` on a later call is the callee's own state changing between calls, not the argument.
That also explains the `call_function` observation: `bHoldHealth: true` proves the function ran, and
nothing on the argument path can turn `0.18` into a value `>= 1`.

### What was actually wrong (and is now fixed)

The audit for parallel numeric implementations found one: the **array-literal** form of a
Vector/Rotator argument narrowed each component through `float` before storing it into an LWC
double-precision struct — `FVector V((float)Arr[0]->AsNumber(), ...)`. So
`{"NewScale3D": [0.18, ...]}` stored `0.18000000715255737` while `{"NewScale3D": {"x": 0.18, ...}}`
stored `0.18` (the object form recurses into the `FDoubleProperty` sub-properties,
`PropertyImport.cpp:848-881`). Silent, no error, and it broke export→import round trips because
vectors are **exported as arrays** (`Utils/PropertyExport.cpp:883-891`). The shared JSON→vector
reader `ReadVectorFieldImpl` (`Utils/JsonUtils.cpp:39-45`) never narrowed; this branch was the
outlier.

### Files changed

- `Plugins/PinWright/Source/PinWright/Private/Utils/PropertyImport.cpp` — dropped the `(float)` casts
  in the array-literal `Vector` and `Rotator` branches (the `LinearColor` casts stay: its components
  really are `float`). Comment records why.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Reflection/TestObjectCallFunctionHandler.cpp` —
  three regression tests, each with an exact (`!=`, not near) comparison:
  - `PinWright.object.call_function.FloatParamArrivesExact` — `SetComponentTickInterval(float)` with
    `0.18, 0.4, 0.2, 0.075, 1.25, 1e-3, -0.18, 2.0, 0.0`; asserts the callee received
    `static_cast<float>(input)` for every one. This is the assertion the ticket asked for.
  - `PinWright.object.call_function.DoubleParamArrivesExact` — object-form vector `(0.18, -0.075, 1e-3)`
    into the `FDoubleProperty` scalar branch.
  - `PinWright.object.call_function.VectorArrayParamKeepsDoublePrecision` — same values, array form.
    Fails on the pre-fix code; pins the fix.

### Reviewer verification

1. Run `PinWright.object.call_function` (three new tests plus the four existing ones). All must pass;
   the two vector tests must agree on the stored value.
2. Counterfactual for the fix: restore the `(float)` casts and confirm
   `VectorArrayParamKeepsDoublePrecision` fails while `DoubleParamArrivesExact` still passes — that
   asymmetry is the defect.
3. Re-check the original field repro **with a read-back of the callee's own inputs**, not of a
   property the callee also writes: call `DebugHoldHealth(0.18)` and read `Health01` in the *same*
   frame, or drive a function whose parameter is stored verbatim. If `Health01` still reads `1`, the
   pawn is restoring it — a host-content question, not a marshalling one.
4. `Content/Python/check_test_ids.py` was run and reports `CLEAN` (4962 ids, no dot-prefix
   collisions).

**Not changed, deliberately** — three adjacent gaps found in the same audit that are *not* this
defect and were left for their own tickets rather than widened into this diff: `PropertyImport.cpp`
has no `FInt8Property` / `FInt16Property` branch though `PropertyExport.cpp:662-671` does (a loud
`Unsupported property type` failure, not a silent one); the string→number branches use
`FCString::Atod`, which is `wcstod` and therefore LC_NUMERIC-dependent (only reachable when a caller
sends a **quoted** number); and the array-element numeric ladder at `PropertyImport.cpp:982-1018` is a
second, weaker copy of the scalar ladder with no unsigned-width or enum branches.

## History
- `#1-filed` `OPEN` reporter — Found while verifying a HUD low-health vignette. `DebugHoldHealth(0.18)`
  left `Health01` at `1` on three separate attempts in one PIE session, while `0.4` and `0.2` landed
  exactly; the callee is a five-node function whose only behaviour is `clamp(Value,0,1)`, so the
  argument is the only variable. Confirmed the function executed by reading back `bHoldHealth: true`.
- `#2-root-cause-refuted-real-defect-fixed` `IN-REVIEW` developer — Traced the whole path by reading
  source, no editor. The filed mechanism does not exist: `args` reaches the handler as one shared
  `FJsonObject` with no re-serialization, the scalar float/double branches assign `AsNumber()`
  directly, and UE's JSON reader rejects a number token with a lost decimal separator loudly rather
  than silently. The ticket's own `property.set` reading of `0.18000000715255737` is a post-write
  **read-back**, i.e. proof the write was exact. The same audit did find a real silent
  fractional-value defect one branch away — the array-literal Vector/Rotator import narrowed LWC
  double components through `float`, so `[0.18,…]` and `{"x":0.18,…}` disagreed and vector
  export→import could not round-trip. Fixed, with three exact-value regression tests. See the
  `## Fix (implemented)` section for evidence, files and reviewer steps. Not compiled here; the
  compile pass is separate.
