---
id: B-niagara-graph-set-parameter-float-clobbered-to-one
title: "niagara.graph.set_parameter clobbers every nonzero Float value to 1.0 (bool extraction overwrites the numeric value)"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, niagara-graph, set-parameter, float, json, silent-wrong-value]
---

# `niagara.graph.set_parameter` writes 1.0 for ANY nonzero Float value

`niagara.graph.set_parameter` accepts a Float `User.*` parameter and a numeric
`value`, reports success, and persists the **wrong value**: every nonzero float
is stored as `1.0` (and `0.0` for any zero/false input). Only an input that
happens to be `0` or `1` round-trips correctly. The success response itself
misreports the stored value (`"value":1`), so the caller has no signal that the
intended value was dropped — it is a silent success-with-wrong-effect.

## Root cause

`NiagaraGraphHandler.cpp` (the `set_parameter` value-extraction block,
~lines 431–448) extracts the payload `value` twice and lets the **bool**
extraction overwrite the numeric one:

```cpp
double NumericValue = 0.0;
if (RawPayload->TryGetNumberField(TEXT("value"), NumericValue))
{
    Val = static_cast<float>(NumericValue);   // Val = 2.5  ✓
    bVal = (NumericValue != 0.0);
}

bool BoolValue = false;
if (RawPayload->TryGetBoolField(TEXT("value"), BoolValue))   // <-- ALSO succeeds for a number
{
    bVal = BoolValue;
    Val = BoolValue ? 1.0f : 0.0f;            // Val = 1.0  ✗  clobbers 2.5
}
```

`FJsonObject::TryGetBoolField` forwards to `FJsonValueNumber::TryGetBool`, which
(engine `Json/Private/Dom/JsonValue.cpp:440`) is:

```cpp
bool FJsonValueNumber::TryGetBool(bool& OutBool) const
{
    OutBool = (Value != 0.0);
    return true;                              // ALWAYS true for any number
}
```

So for any numeric JSON `value`, the second `if` always fires and overwrites the
correctly-parsed float `Val` with `BoolValue ? 1.0f : 0.0f`. The Float branch
that follows (`UserStore.SetParameterValue(Val, FNiagaraVariable(GetFloatDef(), ...))`)
then writes the clobbered `1.0`/`0.0` instead of the input. The bool extraction
must not unconditionally win — it should only apply when the JSON value is
genuinely a boolean (`FJsonValueBoolean`), not a number.

## Repro (replay-confirmed live against `mcp__editor-automation__call`)

System: `/Game/ExampleContent/Niagara/DataChannels/Niagara/NS_SpawnFromIslandNDC`

1. `niagara.add_parameter {scope:"user", name:"User.ReplayFloatKnob", type:"float", defaultValue:1}` → `success:true`.
2. `niagara.graph.set_parameter {assetPath:<system>, parameterName:"User.ReplayFloatKnob", value:2.5}`
   → response `{... "parameterName":"User.ReplayFloatKnob", "value":1}` — sent **2.5**, stored/echoed **1**.
3. `niagara.graph.set_parameter {... value:7.25}` → echoes `"value":1`.
4. `niagara.graph.set_parameter {... value:0.5}`  → echoes `"value":1`.

Every distinct nonzero float (2.5, 7.25, 0.5) is written as `1`. A subsequent
`niagara.inspect` readback shows the user param still at `1`, never the intended
value. (The same task reached the goal only by switching to the type-aware sibling
`niagara.set_parameter scope=user type=float`, which stores 2.5 correctly — so the
Float store and asset path work; the defect is purely this handler's value parse.)

## Impact

This is the RPC the wiki/task names for setting an asset-level Float user default,
and it is unusable for any value other than 0/1: a designer "spawn density" knob
set to 2.5 silently becomes 1.0. The success response actively misreports the
stored value, so an agent that trusts the echo reports the asset as tuned when it
is not. Bool params are unaffected (their value is genuinely bool); integer-valued
0/1 floats round-trip by accident.

**Workaround:** use `niagara.set_parameter` (the type-aware sibling: `scope:"user",
type:"float", value:<x>`), which parses the value per declared type and stores
the float correctly.

**Fix:** Only consult the bool extraction when the JSON value is actually a
boolean. E.g. extract `value` once via `RawPayload->TryGetField("value")` and
branch on `JsonValue->Type == EJson::Boolean` for the bool path; or gate the
`TryGetBoolField` block so it does not run when `TryGetNumberField` already
succeeded. After the fix, the Float branch must write the original `Val`.

## Cross-ref

- `E-niagara-graph-set-parameter-opaque-scope` (OPEN) — same RPC, but a different
  failure mode: that ticket is about the opaque `[PARAM_FAILED]` when a param is
  not an exposed `User.*` Float/Bool (the find/error path). This ticket is the
  value-corruption path: the param IS found, the write "succeeds," but the stored
  Float value is wrong. Not a duplicate — different code path, different symptom.
- `B-set-niagara-param-no-validation` (IN-REVIEW) — the runtime *component-side*
  setters' silent-success-for-bogus-name defect; distinct method and direction.

## History
- `#2-fix` `IN-REVIEW` developer — Fixed the double-extraction in `NiagaraGraphHandler.cpp` (`niagara.graph.set_parameter`). Replaced the unconditional numeric-then-bool extraction with a single `RawPayload->TryGetField("value")` and a branch on `ValueField->Type == EJson::Boolean`: the bool path (`Val = bVal ? 1.0f : 0.0f`) now fires ONLY for a genuine JSON boolean; any number goes through `TryGetNumber` and keeps its exact float `Val`. This removes the `FJsonValueNumber::TryGetBool`-always-true clobber, so a Float user param set to 2.5 now stores/echoes 2.5. File: `Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/NiagaraGraphHandler.cpp` (value-extraction block ~431–455). Regression tests added in `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestNiagaraHandlers.cpp`: `FNiagaraGraphSetParameterFloatRoundTripTest` (`...set_parameter.FloatRoundTrip`) creates a transient system, exposes a Float `User.*` param (default 9.0), invokes the handler with value 2.5, and asserts BOTH the echoed `value` and the value read back via `GetExposedParameters().GetParameterValue<float>(...)` equal 2.5 — it fails if the clobber is reintroduced; plus `FNiagaraGraphSetParameterBoolRoundTripTest` (`...set_parameter.BoolRoundTrip`) guards that a genuine JSON `true` still resolves through the Bool store path. Not compiled/run here (later phase).
- `#1-initial-repro` `OPEN` reporter — Seed-mode task on `niagara.graph.set_parameter` (tune NS_SpawnFromIslandNDC's spawn knob to 2.5). Replay-confirmed live: added `User.ReplayFloatKnob` (float, default 1) via `niagara.add_parameter`, then `niagara.graph.set_parameter value:2.5` echoed `"value":1`; retries with 7.25 and 0.5 both echoed `"value":1` — every nonzero float clobbered to 1.0. Root cause in `NiagaraGraphHandler.cpp` value-extraction block (~431–448): the numeric extraction sets `Val` correctly, then `TryGetBoolField("value", ...)` ALSO succeeds for any number (engine `JsonValue.cpp:440` `FJsonValueNumber::TryGetBool` always returns true, `OutBool = Value != 0.0`) and overwrites `Val = BoolValue ? 1.0f : 0.0f`, so the Float branch writes 1.0. Silent success-with-wrong-effect; the success response misreports `"value":1`. Workaround/correct path: `niagara.set_parameter` (type-aware) stores 2.5 fine. Ripgrep across OPEN/closed found no existing ticket on the float-clobber (E-niagara-graph-set-parameter-opaque-scope covers only the find/error path of the same RPC; not a duplicate).
</content>
</invoke>
