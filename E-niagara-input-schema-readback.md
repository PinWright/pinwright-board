---
id: E-niagara-input-schema-readback
title: "niagara: typed per-input schema introspection for module stack inputs"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [niagara, introspection, schema, parity-ue58]
---

# niagara: typed per-input schema introspection for module stack inputs

Agents setting module inputs must guess input names, types, and value modes because the
per-module readback carries no typed input schema. The only per-module readback,
`niagara.inspect` includeStack (built by `NiagaraDumpBuilder::BuildStackModuleJson`,
NiagaraDumpBuilder.cpp:907), emits module identity + `staticSwitchInputs` only — NOT the
regular stack inputs with their types, value modes, and current values. This causes
trial-and-error loops against `set_module_input`.

Note the gap is specifically the absence of a **structured** per-placed-module input schema,
not a total absence of readback: `niagara.decompile_nir` already resolves each input's value
mode (literal / `$linked-parameter` / dynamic-input chain / static-switch) but only as
unstructured NIR **text**, and `niagara.graph.get` surfaces the module SCRIPT's declared
signature (name/type) but not the placed module's current binding. There is no single
structured JSON that lists a placed module's inputs with type + value-mode + current value,
so callers parse NIR text or hand-assemble graph pins.

UE 5.8 parity evidence: NiagaraToolsets exposes typed per-input JSON schemas with `oneOf`
value shapes plus `GetSystemSchema` / `GetEmitterTopology` / `GetDynamicInputChain`
introspection (new in the 5.8.0 release build under `C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\NiagaraToolsets\`).
That structured introspection is what makes Epic's Niagara surface reliable for agents.

Fix (extend, do NOT add a parallel verb — dual-surface convention + the original dedup note):
- Extend `BuildStackModuleJson` — the shared builder feeding `niagara.inspect` includeStack
  and the `niagara_stack.json` dump sidecar — with a `moduleInputs` array. Each entry:
  `name`, `type` (+ canonical `typeInfo`), `valueMode` (default/local/linked/dynamicInput),
  the current `value` / `linkedParameter` / `dynamicInput` script, and `enumOptions` for enum
  inputs. Enumerate the module's stack inputs via
  `FNiagaraStackGraphUtilities::GetStackFunctionInputs` (EnumerateScriptInputs surfaces only
  the ParameterMap input node, not the `Module.*` inputs) and classify each value mode by
  walking the override node (`NiagaraResetModuleInput::FindStackFunctionOverrideNode`) —
  handling BOTH linked-parameter wirings (classic `UNiagaraNodeInput` and modern
  `UNiagaraNodeParameterMapGet`). Bump the `niagara_stack.json` aspect version.
- Coordinate with the deferred emitter/entryId single-module stack projection carved out in
  `E-niagara-inspect-no-param-readback-projection` (that is the response-SIZE concern; this is
  the orthogonal CONTENT concern). A large share of the cited `set_module_input`
  trial-and-error is also driven by the separate bug `B-niagara-module-input-stack-infer`.

Acceptance: `niagara.inspect` includeStack on a system with a stock SpawnRate module lists
that module's `moduleInputs` including the `SpawnRate` input with its float type and current
value mode, matching the editor UI.

## History
- `#1-inputs-opaque` `OPEN` reporter — Module stack inputs have no schema/type/value-mode readback, forcing guesswork in set_module_input. Epic 5.8 ships typed per-input schemas + GetDynamicInputChain; add get_module_inputs (dedup with any existing readback first).
- `#2-reword-extend-builder` `IN-REVIEW` developer — REWORD: reproduced (no structured per-input schema in the module readback), but dropped the overstated "no readback" premise (decompile_nir already gives value modes as NIR text) and the parallel-verb proposal (dual-surface rule + the dedup note). Implemented by extending the shared stack builder, not a new verb: added `moduleInputs` to `NiagaraDumpBuilder::BuildStackModuleJson` (name/type/typeInfo/valueMode/value/linkedParameter/dynamicInput/enumOptions) via new `NiagaraDumpBuilder::BuildModuleInputsJson`. Declared inputs come from new `NiagaraEdit::EnumerateModuleStackInputs` (wraps `FNiagaraStackGraphUtilities::GetStackFunctionInputs`, ModuleInputsOnly — EnumerateScriptInputs only surfaces the ParameterMap input node); value modes from new `NiagaraEdit::ClassifyModuleInputBindings` (walks `FindStackFunctionOverrideNode`, classifying literal / linked (both `UNiagaraNodeInput` and `UNiagaraNodeParameterMapGet` wirings) / dynamic-input). Short input names via `FNiagaraParameterHandle`. Bumped `niagara_stack.json` aspect version to 2. Files: NiagaraDumpBuilder.cpp/.h, NiagaraEditTypes.cpp/.h, AssetDumpCache.cpp. Test: PinWright.niagara.ModuleInputsSchema (Tests/Niagara/TestNiagaraGetModuleInputs.cpp) — lists SpawnRate typed, and reports valueMode "linked"→User.Speed after binding; differential-verified (pre-fix tree fails to compile without BuildModuleInputsJson).
