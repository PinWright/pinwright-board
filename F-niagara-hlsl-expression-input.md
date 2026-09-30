---
id: F-niagara-hlsl-expression-input
title: "niagara.set_module_input: HLSL-expression value mode"
status: OPEN
severity: Low
category: feature
tags: [niagara, authoring, hlsl, parity-ue58, gap-analysis-2026-09-30]
blockedBy: [F-niagara-create-module-script]
---

# niagara.set_module_input: HLSL-expression value mode

Split out from **F-niagara-dynamic-input-authoring** (which delivers the dynamic-input value mode). `niagara.set_module_input` has no inline-HLSL value mode: an author cannot set a module input to a custom HLSL expression node — the power-user escape hatch alongside literals, linked parameters, and dynamic-input chains. A `{ "expression": "hlsl..." }` value currently falls through to the literal `InferNiagaraInputType` path and is rejected `UNSUPPORTED_INPUT_VALUE` (`Handlers\Niagara\NiagaraEditHandler.cpp` :882).

Proposed scope:
- Extend `set_module_input` value shapes with `{ "expression": "hlsl..." }` — assign a custom-HLSL dynamic-input node to the module input's override pin.

Implementation note / why separate from the dynamic-input mode: the engine setter `FNiagaraStackGraphUtilities::SetCustomExpressionForFunctionInput` (`NiagaraStackGraphUtilities.h` ~:235) is NOT `NIAGARAEDITOR_API`-exported (unlike the dynamic-input peer `SetDynamicInputForFunctionInput` at ~:233 which IS), so this needs the reflection / non-exported-API workaround the module already uses for CustomHlsl writes elsewhere (`NiagaraGraphHandler.cpp` custom-HLSL path). That makes it materially higher risk than the dynamic-input mode, which is why it is its own lower-priority ticket.

Acceptance: set an HLSL-expression input on a scalar module input; the override pin is driven by a `UNiagaraNodeCustomHlsl` node and `niagara.compile` passes.

## History
- `#1-split-from-dynamic-input` `OPEN` reporter — Split from F-niagara-dynamic-input-authoring so the load-bearing dynamic-input value mode ships independently. The HLSL `{expression}` mode is a rarer power-user path whose engine setter (`SetCustomExpressionForFunctionInput`) is non-exported (reflection needed), so it is tracked separately at lower priority.
- `#2-deferred-until-create-module-script` `OPEN` reporter — Deferred until F-niagara-create-module-script lands (blockedBy set). Both need the same CustomHlsl machinery: a typed `Signature` filled before `Finalize` (pins come from `Signature`, `NiagaraNodeFunctionCall.cpp:465`, not from the HLSL text) plus the reflected `CustomHlsl` write, because `SetCustomHlsl` is still not exported on 5.8 (`NiagaraNodeCustomHlsl.h:13` MinimalAPI, :19-20). `FNiagaraStackGraphUtilities::SetCustomExpressionForFunctionInput` is also still unexported on 5.8 (`NiagaraStackGraphUtilities.h:237`, while `SetDynamicInputForFunctionInput` at :235 is exported). Parity context from the 2026-09-30 gap analysis: Epic is the only competitor with this feature, as a single-rvalue inline expression on a stack input (`FNiagaraExt_StackInputData_HlslExpression`, `NiagaraExternalSystemEditorUtilities.h:583`, applied at `.cpp:2999`; 5.8-only toolset). Severity stays Low.
