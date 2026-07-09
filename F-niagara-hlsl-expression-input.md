---
id: F-niagara-hlsl-expression-input
title: "niagara.set_module_input: HLSL-expression value mode"
status: OPEN
severity: Low
category: feature
tags: [niagara, authoring, hlsl, parity-ue58]
---

# niagara.set_module_input: HLSL-expression value mode

Split out from **F-niagara-dynamic-input-authoring** (which delivers the dynamic-input value mode). `niagara.set_module_input` has no inline-HLSL value mode: an author cannot set a module input to a custom HLSL expression node — the power-user escape hatch alongside literals, linked parameters, and dynamic-input chains. A `{ "expression": "hlsl..." }` value currently falls through to the literal `InferNiagaraInputType` path and is rejected `UNSUPPORTED_INPUT_VALUE` (`Handlers\Niagara\NiagaraEditHandler.cpp` :882).

Proposed scope:
- Extend `set_module_input` value shapes with `{ "expression": "hlsl..." }` — assign a custom-HLSL dynamic-input node to the module input's override pin.

Implementation note / why separate from the dynamic-input mode: the engine setter `FNiagaraStackGraphUtilities::SetCustomExpressionForFunctionInput` (`NiagaraStackGraphUtilities.h` ~:235) is NOT `NIAGARAEDITOR_API`-exported (unlike the dynamic-input peer `SetDynamicInputForFunctionInput` at ~:233 which IS), so this needs the reflection / non-exported-API workaround the module already uses for CustomHlsl writes elsewhere (`NiagaraGraphHandler.cpp` custom-HLSL path). That makes it materially higher risk than the dynamic-input mode, which is why it is its own lower-priority ticket.

Acceptance: set an HLSL-expression input on a scalar module input; the override pin is driven by a `UNiagaraNodeCustomHlsl` node and `niagara.compile` passes.

## History
- `#1-split-from-dynamic-input` `OPEN` reporter — Split from F-niagara-dynamic-input-authoring so the load-bearing dynamic-input value mode ships independently. The HLSL `{expression}` mode is a rarer power-user path whose engine setter (`SetCustomExpressionForFunctionInput`) is non-exported (reflection needed), so it is tracked separately at lower priority.
