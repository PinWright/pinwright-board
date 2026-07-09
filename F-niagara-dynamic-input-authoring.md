---
id: F-niagara-dynamic-input-authoring
title: "niagara.set_module_input: assign dynamic-input chains to module inputs"
status: IN-REVIEW
severity: Medium
category: feature
tags: [niagara, authoring, dynamic-input, parity-ue58]
---

# niagara.set_module_input: assign dynamic-input chains to module inputs

`niagara.set_module_input` supports literal values and linked-attribute/parameter binds only (`Handlers\Niagara\NiagaraEditHandler.cpp` SetModuleInput branch ~:757-905; linked path via `TryGetLinkedParameterRequest` :1375). The code can CLEAR an existing dynamic-input chain (comment near :801, `NiagaraResetModuleInput::ClearModuleInputOverride`) but cannot create or assign one: a `{ "dynamicInput": "<path>" }` value matches no link key, falls through to the literal `InferNiagaraInputType` path, and is rejected `UNSUPPORTED_INPUT_VALUE` (:882). Dynamic inputs are how most real Niagara content parameterizes module inputs (curves over life, random ranges, multiply chains), so agents currently cannot author idiomatic effects — this is the write-side peer of the linked-parameter value mode that F-niagara-link-module-input-to-parameter (IN-REVIEW) added.

Scope (this ticket — the dynamic-input value mode only):
- Extend `set_module_input` value shapes with `{ "dynamicInput": "ScriptAssetPath", "inputs": {...} }`. Recursive: nested `inputs` set literals or further dynamic inputs on the assigned dynamic-input node. The engine primitive is `FNiagaraStackGraphUtilities::SetDynamicInputForFunctionInput` (NIAGARAEDITOR_API-exported), wired into the same `GetOrCreateStackFunctionInputOverridePin` override-pin seam the linked-parameter path uses.

Out of scope (split out during reword):
- HLSL-expression value mode `{ "expression": "hlsl..." }` -> tracked in **F-niagara-hlsl-expression-input**. Its engine setter `SetCustomExpressionForFunctionInput` is NOT NIAGARAEDITOR_API-exported (needs the reflection workaround), so it is a separate, higher-risk deliverable.
- Chain readback in the module-input info path -> owned by **E-niagara-input-schema-readback** (`get_module_inputs`). Dynamic-input readback already exists on the model/dump side (`NiagaraModelReferences.cpp` `BuildDynamicInputNodeJson`/`BuildNestedDynamicInputs`, valueMode "dynamic_input"; `NiagaraModelBuilder.h` `BuildDynamicInputModel`), so no read-side work belongs here.

UE 5.8 parity evidence: NiagaraToolsets `SetStackInputData` (`C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\NiagaraToolsets\`) treats the dynamic-input chain as a first-class value mode alongside Local / Linked-attribute.

Acceptance: assign a float dynamic-input script (e.g. `/Niagara/DynamicInputs/Add/Add_Float`) to a float module input (e.g. SpawnRate's `SpawnRate`), optionally with a nested input set; the call succeeds and the module input's override pin is driven by a dynamic-input function-call node running that script (structural), not a literal default.

## History
- `#2-reword-dynamic-input-only` `IN-REVIEW` developer — Reworded to the dynamic-input value mode only: dropped the redundant readback bullet (owned by E-niagara-input-schema-readback, and already present on the model side — `NiagaraModelReferences.cpp` BuildDynamicInputNodeJson/BuildNestedDynamicInputs); split the HLSL `{expression}` mode to F-niagara-hlsl-expression-input (its engine setter `SetCustomExpressionForFunctionInput` is non-exported vs the exported `SetDynamicInputForFunctionInput`, materially higher risk). Severity High -> Medium to match the sibling authoring value modes (F-niagara-link, F-niagara-curve). Implemented `{ dynamicInput, inputs }` on `niagara.set_module_input`: detect the request (`TryGetDynamicInputRequest`), load+validate the DynamicInput-usage script, clear any prior override (`NiagaraResetModuleInput::ClearModuleInputOverride`), create a fresh override pin, and assign the dynamic-input node via `FNiagaraStackGraphUtilities::SetDynamicInputForFunctionInput`, recursing for nested `inputs` (literals or nested dynamic inputs, depth-guarded). Handler now reports `dynamicInput: "<path>"`. Files: `Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp` (helpers + branch + result + value param doc). Tests (`Tests/Niagara/TestNiagaraSetModuleInputDynamicInput.cpp`): `PinWright.niagara.set_module_input.AssignsDynamicInputChain` (adopted red test — flat assignment) and `PinWright.niagara.set_module_input.AssignsDynamicInputChainWithNestedInput` (recursive nested-input coverage). Compile + green driven by the workflow's later phases.
- `#1-no-dynamic-input-create` `OPEN` reporter — set_module_input can clear but not create dynamic-input chains, and has no HLSL expression mode (NiagaraEditHandler.cpp evidence). Epic 5.8 SetStackInputData treats both as first-class value modes; extend value shapes accordingly.
