---
id: F-niagara-dynamic-input-nested-inputs
title: "niagara.set_module_input: recursive nested-input authoring on an assigned dynamic input"
status: OPEN
severity: Low
category: feature
tags: [niagara, authoring, dynamic-input, parity-ue58]
---

# niagara.set_module_input: recursive nested-input authoring on an assigned dynamic input

Split out from **F-niagara-dynamic-input-authoring** (which delivered assigning a dynamic input to a module input via `{ dynamicInput: "<path>" }`). That leaves the assigned dynamic input at its in-script default inputs. Many real dynamic inputs must have their OWN inputs set to be useful (a Curve-over-Life needs its curve; an Add/Multiply needs its addend/factor). This ticket adds recursive nested-input authoring:

```
value: { "dynamicInput": "<ScriptAssetPath>", "inputs": { "<inputName>": <literal | { dynamicInput, inputs }> } }
```

so a chain can be authored in one call.

Implementation note / why separate: setting a nested input requires the assigned dynamic-input node's authorable stack inputs (their names AND types) to resolve `inputName` and type the nested override pin. `NiagaraEdit::EnumerateScriptInputs` does NOT provide these — it surfaces only the script's parameter-map input node (observed: Add_Float enumerates a single `NewInput` of type `NiagaraParameterMap`, not its value inputs). The correct source is `FNiagaraStackGraphUtilities::GetStackFunctionInputs` (NIAGARAEDITOR_API) which needs an `FCompileConstantResolver` built from the target system/emitter + script usage. That machinery (and a nested-input regression test that discovers a real input name via the same resolver) is the work this ticket tracks.

Acceptance: assign a dynamic input with a nested input set (e.g. `{ dynamicInput: "Add_Float", inputs: { "<A>": 42.0 } }`); the call succeeds, the module input override pin is driven by the dynamic-input node, and the named nested input carries the set literal (or a further nested dynamic-input node). Depth-guard runaway/cyclic chains.

## History
- `#1-split-from-dynamic-input` `OPEN` reporter — Split from F-niagara-dynamic-input-authoring during implementation: the flat assignment shipped, but recursive nested `inputs` needs resolver-based stack-input enumeration (`GetStackFunctionInputs` + `FCompileConstantResolver`) that `EnumerateScriptInputs` cannot supply, plus its own test, so it is tracked separately at lower priority.
