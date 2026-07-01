---
id: B-niagara-static-switch-not-decoded
title: "Niagara static switch resolved values not decoded — UNiagaraNodeStaticSwitch branches invisible in the model"
status: DONE
severity: Medium
category: bug
tags: [niagara, static-switch, asset-dump, coverage-gap]
---

# `UNiagaraNodeStaticSwitch` resolved value is not in the IR

Niagara's most common branch-selection mechanism — `UNiagaraNodeStaticSwitch`
— lets module authors expose a "static switch" input that picks one
branch of behavior at compile time (e.g. choose between curl noise
and simple drag, or between additive and multiplicative blending).

The IR captures static switch nodes only as **generic UEdGraphNode
entries in `niagara_graphs.json`** — class path, GUID, pins. The
*selected branch* (the resolved switch value, the user's choice in
the stack) is **not decoded** anywhere:
- Not in `niagara_stack.json`
- Not in `niagara_model.json`'s function-call records
- Not in `niagara_parameters.json` (static switches use a different
  storage mechanism than runtime parameters)

There is no `staticSwitchInputs` array equivalent to what the
Niagara stack UI shows.

## Why it matters

Static switches are everywhere in Niagara modules. Without the
resolved value:
- A consumer reading the IR cannot tell which branch of a module
  is active.
- `niagara.set_property` cannot target the switch.
- Any authoring workflow that needs to flip a switch on / off must
  manipulate raw graph nodes by GUID, defeating the point of the
  IR.

## Fix

1. **Dump:** in the function-call record under
   `niagara_stack.json` and `niagara_model.json`, add a
   `staticSwitchInputs[]` array with `{ name, type, value }`
   for each static switch input.
2. **Authoring RPC:** add `niagara.set_static_switch` with
   targeting (system / emitter / module / switch name) +
   value. Implementation goes through
   `UNiagaraStackFunctionInputBinder` or equivalent.

The static switch data exists on the function-call binding (not
on the switch node itself) via the rapid-iteration parameter
override mechanism. The dumper must read both the resolved value
on the binding and the switch node's declared default.

## Repro

Any NiagaraSystem using a built-in module with a static switch
(common: `Apply Initial Mesh Orientation`, `Sample Skeletal Mesh`).
Inspect `niagara_stack.json` for the relevant module — no static
switch values present.

## History
- `#1-initial-spec` `OPEN` reporter — Niagara coverage parity audit found `UNiagaraNodeStaticSwitch` is captured only as a raw graph node by GUID; the resolved branch value (what the stack UI shows the user picking) is not decoded into any IR aspect. Common feature in Niagara modules; without it, the IR can't represent which branch of a module is active. Add `staticSwitchInputs[]` to function-call records + `niagara.set_static_switch` authoring RPC.
- `#2-decoded-switch-and-set-rpc` `IN-REVIEW` developer — Added NiagaraDumpBuilder::BuildStaticSwitchInputs helper that decodes UNiagaraNodeStaticSwitch resolved values from the function-call script's RapidIterationParameters; wired into BuildStackModuleJson (NiagaraDumpBuilder.cpp) and BuildStackModuleModel (NiagaraModelBuilder.cpp) so staticSwitchInputs[] now appears on niagara_stack.json and niagara_model.json. Added niagara.set_static_switch RPC in NiagaraEditHandler.cpp mirroring niagara.set_module_input targeting. Regression test at TestNiagaraDumpStaticSwitch.cpp asserts the helper is exported and emits a present staticSwitchInputs key.
- `#3-corrected-engine-api-note` `IN-REVIEW` developer — Correction to entry #2: static switch resolved values flow through the caller-side input pin DefaultValue (resolved by FindStaticSwitchInputPin), NOT through the function-call script's RapidIterationParameters as #2 stated. The implemented code in BuildStaticSwitchInputs and niagara.set_static_switch reads/writes the caller pin; the prior wording was inherited from the planning phase before engine source inspection. Behavior is unchanged; only the doc note is corrected.
- `#4-verify-static-switch-dump` `DONE` tester — Verified: `call("niagara")` lists `niagara.set_static_switch`, and `asset.dump` on `/Game/Effects/Particles/Weapons/NS_WeaponFire.NS_WeaponFire` wrote Niagara aspects where `niagara_stack.json` had 166 module entries with non-empty `staticSwitchInputs` and `niagara_model.json` had 42 non-empty `staticSwitchInputs` arrays; sample stack record was `SystemState` input `Inactive Response` with `type: "Enum"` and `value: 1`.
