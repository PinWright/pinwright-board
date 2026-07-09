---
id: F-niagara-dynamic-input-authoring
title: "niagara.set_module_input: assign a dynamic input to a module input"
status: IN-REVIEW
severity: Medium
category: feature
tags: [niagara, authoring, dynamic-input, parity-ue58]
---

# niagara.set_module_input: assign a dynamic input to a module input

`niagara.set_module_input` supported literal values and linked-attribute/parameter binds only (`Handlers\Niagara\NiagaraEditHandler.cpp` SetModuleInput branch ~:757-905; linked path via `TryGetLinkedParameterRequest`). The code could CLEAR an existing dynamic-input chain (`NiagaraResetModuleInput::ClearModuleInputOverride`) but could not create or assign one: a `{ "dynamicInput": "<path>" }` value matched no link key, fell through to the literal `InferNiagaraInputType` path, and was rejected `UNSUPPORTED_INPUT_VALUE` (:882). Dynamic inputs are how most real Niagara content parameterizes module inputs (curves over life, random ranges, multiply chains), so agents could not author idiomatic effects — this is the write-side peer of the linked-parameter value mode that F-niagara-link-module-input-to-parameter added.

Scope (this ticket — assign a dynamic input to a module input):
- Extend `set_module_input` value shapes with `{ "dynamicInput": "ScriptAssetPath" }`. Assigns the dynamic-input script's function-call node onto the module input's override pin via `FNiagaraStackGraphUtilities::SetDynamicInputForFunctionInput` (NIAGARAEDITOR_API-exported), the same override-pin seam the linked-parameter path uses.

Out of scope (split out):
- Recursive nested-input authoring `{ dynamicInput, inputs: {...} }` (setting the assigned dynamic input's own inputs) -> **F-niagara-dynamic-input-nested-inputs**. It needs resolver-based stack-input enumeration (`GetStackFunctionInputs` + `FCompileConstantResolver`); `EnumerateScriptInputs` only surfaces the script's parameter-map pin, not the authorable inputs, so nested type/name resolution cannot be done cleanly here.
- HLSL-expression value mode `{ "expression": "hlsl..." }` -> **F-niagara-hlsl-expression-input** (its engine setter `SetCustomExpressionForFunctionInput` is not exported, needs the reflection workaround).
- Chain readback in the module-input info path -> owned by **E-niagara-input-schema-readback**. Dynamic-input readback already exists on the model/dump side (`NiagaraModelReferences.cpp` `BuildDynamicInputNodeJson`/`BuildNestedDynamicInputs`, valueMode "dynamic_input").

Acceptance: assign a float dynamic-input script (e.g. `/Niagara/DynamicInputs/Add/Add_Float`) to a float module input (e.g. SpawnRate's `SpawnRate`); the call succeeds and the module input's override pin is driven by a dynamic-input function-call node running that script (structural), not a literal default.

## History
- `#2-dynamic-input-assign` `IN-REVIEW` developer — Reworded to a single load-bearing deliverable and implemented it. Dropped the redundant readback bullet (owned by E-niagara-input-schema-readback, already present on the model side); split the HLSL `{expression}` mode to F-niagara-hlsl-expression-input (non-exported setter); split recursive nested `inputs` authoring to F-niagara-dynamic-input-nested-inputs (discovered during implementation: `EnumerateScriptInputs` surfaces only the parameter-map pin, so nested-input type/name resolution needs `GetStackFunctionInputs`+`FCompileConstantResolver` and its own test — out of scope here). Severity High -> Medium to match the sibling authoring value modes (F-niagara-link, F-niagara-curve). Implemented `{ dynamicInput: "<path>" }` on `niagara.set_module_input`: detect the request (`TryGetDynamicInputRequest`), load+validate the DynamicInput-usage script (`LoadDynamicInputScript`), determine the override-pin type from the dynamic input's own value output node (`GetDynamicInputOutputType` — the type the pin actually carries; `EnumerateScriptInputs`/`FindModuleInputDeclaredType` cannot supply a module input's type), clear any prior override (`NiagaraResetModuleInput::ClearModuleInputOverride`), create the override pin, and assign the node via `FNiagaraStackGraphUtilities::SetDynamicInputForFunctionInput`. Handler now reports `dynamicInput: "<path>"`; `value` param doc updated. Files: `Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp`. Regression test (adopted red test, now green): `PinWright.niagara.set_module_input.AssignsDynamicInputChain` (`Tests/Niagara/TestNiagaraSetModuleInputDynamicInput.cpp`) — drives the production RPC with `{ dynamicInput: "Add_Float" }` on SpawnRate's `SpawnRate` and asserts the override pin is driven by a dynamic-input node running that script. Compiles clean; red test observed failing pre-fix (UNSUPPORTED_INPUT_VALUE) and passing post-fix.
- `#1-no-dynamic-input-create` `OPEN` reporter — set_module_input can clear but not create dynamic-input chains, and has no HLSL expression mode (NiagaraEditHandler.cpp evidence). Epic 5.8 SetStackInputData treats both as first-class value modes; extend value shapes accordingly.
