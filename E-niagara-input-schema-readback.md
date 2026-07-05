---
id: E-niagara-input-schema-readback
title: "niagara: typed per-input schema introspection for module stack inputs"
status: OPEN
severity: Medium
category: ergonomic
tags: [niagara, introspection, schema, parity-ue58]
---

# niagara: typed per-input schema introspection for module stack inputs

Agents setting module inputs must guess input names, types, and allowed value modes; there is no schema readback for a module's stack inputs (type, enum choices, whether a dynamic-input chain or linked parameter is currently assigned). This causes trial-and-error loops against `set_module_input`.

UE 5.8 parity evidence: NiagaraToolsets exposes typed per-input JSON schemas with `oneOf` value shapes plus `GetSystemSchema` / `GetEmitterTopology` / `GetDynamicInputChain` introspection (new in the 5.8.0 release build of `C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\NiagaraToolsets\`). That introspection is what makes Epic's Niagara surface unusually reliable for agents despite covering less than PinWright's.

Proposed scope:
- `niagara.get_module_inputs(asset, emitter, module)` returning per-input: name, type, current value mode (local/linked/dynamic-input/data-interface), current value or chain summary, enum options, defaults.
- Note: developer must dedup against any existing module-info readback in the namespace at implementation time; extend rather than add a parallel method if one exists.

Acceptance: for a stock Fountain emitter module, returned schema lists every visible stack input with correct types and current values matching the editor UI.

## History
- `#1-inputs-opaque` `OPEN` reporter — Module stack inputs have no schema/type/value-mode readback, forcing guesswork in set_module_input. Epic 5.8 ships typed per-input schemas + GetDynamicInputChain; add get_module_inputs (dedup with any existing readback first).
