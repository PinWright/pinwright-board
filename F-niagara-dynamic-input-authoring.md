---
id: F-niagara-dynamic-input-authoring
title: "niagara.set_module_input: dynamic-input chains and HLSL expression value modes"
status: OPEN
severity: High
category: feature
tags: [niagara, authoring, dynamic-input, hlsl, parity-ue58]
---

# niagara.set_module_input: dynamic-input chains and HLSL expression value modes

`niagara.set_module_input` supports literal values and linked-attribute/parameter binds only (`Handlers\Niagara\NiagaraEditHandler.cpp` ~:757-950; `TryGetLinkedParameterRequest` at :1375). The code can CLEAR an existing dynamic-input chain (comment near :801) but cannot create or assign one, and there is no HLSL-expression value mode. Dynamic inputs are how most real Niagara content parameterizes module inputs (curves over life, random ranges, multiply chains), so agents currently cannot author idiomatic effects.

UE 5.8 parity evidence: NiagaraToolsets (61 C++ tools, `C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\NiagaraToolsets\Source\NiagaraToolsets\Private\NiagaraToolset_System.cpp` and siblings) `SetStackInputData` supports Local / Linked-attribute / HLSL expression / DataInterface / DynamicInput-chain value modes as first-class, plus `GetDynamicInputChain` readback.

Proposed scope:
- Extend `set_module_input` value shapes: `{ "dynamicInput": "ScriptAssetPath", "inputs": {...} }` (recursive chain) and `{ "expression": "hlsl..." }`.
- Chain readback in the module-input info path (pairs with E-niagara-input-schema-readback).

Acceptance: assign a Curve-over-Life dynamic input to a float module input with nested inputs set, verify via readback and NIR/dump sidecar; set an HLSL expression input and confirm compile passes via `niagara.compile`.

## History
- `#1-no-dynamic-input-create` `OPEN` reporter — set_module_input can clear but not create dynamic-input chains, and has no HLSL expression mode (NiagaraEditHandler.cpp evidence). Epic 5.8 SetStackInputData treats both as first-class value modes; extend value shapes accordingly.
