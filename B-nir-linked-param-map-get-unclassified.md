---
id: B-nir-linked-param-map-get-unclassified
title: "NIR override-chain resolver does not recognise a Parameter Map Get upstream, so every linked-parameter module input renders as the override pin's literal instead of `$Namespace.Name`"
status: IN-REVIEW
severity: Medium
category: bug
tags: [gap-analysis-2026-09-28, niagara, nir, decompile, override-chain, linked-parameter]
encounters: 1
lastSeen: 2026-09-29T18:18:36Z
---

# Linked parameters are classified by the wrong upstream node class

`EmitInputValueExpr` (`Source/PinWright/Private/NIR/NIRTextEmitter.cpp`) treats only an upstream
`UNiagaraNodeInput` as a linked parameter. The engine never wires a linked parameter that way:
`FNiagaraStackGraphUtilities::SetLinkedParameterValueForFunctionInput` always creates a
`UNiagaraNodeParameterMapGet` whose output pin is named for the bound parameter
(`C:\UE_5.8\Engine\Plugins\FX\Niagara\Source\NiagaraEditor\Private\ViewModels\Stack\NiagaraStackGraphUtilities.cpp:2158-2159`).
Production's own classifier already knows this: `Handlers/Niagara/NiagaraEditTypes.cpp:2282-2325` maps
a Map Get upstream to `valueMode: "linked"` and notes that an upstream `UNiagaraNodeInput` means a
data-interface or object value. A Map Get therefore fell through to NIR's "unclassified upstream node"
fallback. That fallback adds a `Warnings` entry and prints the override pin's literal default (usually
`<empty>`), so `niagara.decompile_nir` and `nir.txt` report the wrong value for every linked
module input.

Masked until now by `B-nir-stack-omits-override-only-module-inputs`, which kept regular-input
overrides from reaching this function at all. `PinWright.niagara.decompile_nir.OverrideLinkedParam`
(`TestNIRDecompiler.cpp:1082`, first real run in
`Saved/PinWright/test-runs/7b2d4d4c5a494988a2c90b28389aad29/automation.log`) asserts
`input SpawnRate = $User.Speed`. Its fixture links through the engine helper, so it needs both fixes.

## History
- `#1-map-get-not-classified` `OPEN` reporter — `EmitInputValueExpr` recognises linked parameters only via `UNiagaraNodeInput`, but `SetLinkedParameterValueForFunctionInput` always creates a `UNiagaraNodeParameterMapGet`. Linked inputs fall to the unclassified-node fallback and print the pin literal. Found while tracing `OverrideLinkedParam`'s first real failure.
- `#2-classify-map-get-as-link` `IN-REVIEW` developer — Added a case to `EmitInputValueExpr` after the `UNiagaraNodeInput` case: an upstream node that `IsA` `/Script/NiagaraEditor.NiagaraNodeParameterMapGet` (resolved by `FindObject`; the class has no `NIAGARAEDITOR_API`, same pattern as `NIRGraphEmitter_Dataflow.cpp`) returns `FormatParameterRef(UpstreamPin->PinName)`, i.e. `$User.Speed`. The linked-parameter bullet in `docs/wiki-src/niagara.nir.md` is corrected. `-SingleFile` compile of `NIRTextEmitter.cpp` on UE 5.8: `Result: Succeeded`. Not run: verification is the next run of `PinWright.niagara.decompile_nir.OverrideLinkedParam`.
