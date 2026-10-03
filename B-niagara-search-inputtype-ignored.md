---
id: B-niagara-search-inputtype-ignored
title: "niagara.search_modules accepts inputType but never filters on it"
status: DONE
severity: Medium
category: bug
tags: [niagara, search-modules, input-type, accepted-and-ignored, false-success]
---

# A documented discovery filter is absent from the implementation

`niagara.search_modules` declares optional `inputType` as the DynamicInput type filter
(`NiagaraSearchHandler.cpp:402-411`). The body reads `query`, `stage`, `sourceFilter`, and `limit`
(`:421-428`) and filters usage, stage, source, and score (`:461-505`), but never reads
`inputType`. Its returned `inputs` and `outputs` arrays are explicitly empty placeholders
(`:563-565`). The verb still returns normal results and `totalMatches` success.

A caller asking for Dynamic Inputs compatible with `Niagara.Float` therefore receives the same
set as one asking for `Niagara.Vector`, with no warning that the filter was ignored. The DONE
feature ticket said signature emission was deferred, but the live RPC kept advertising the filter;
deferred implementation must not look like an applied constraint.

## What it should do

Either load/inspect DynamicInput signatures and apply the filter, or remove/refuse `inputType`
until that exists. The regression should use two types with different result sets and assert that
the filter changes `totalMatches` and every returned output type matches.

## Workaround

Omit `inputType`, inspect each candidate separately, and filter client-side.

## Related

- `F-search-api-niagara-modules`
- `E-niagara-create-node-input-type-unverifiable`

## History
- `#1-source-scan` `OPEN` reporter -- Whole-file literal scan found the declaration as the only
  `inputType` occurrence; the full handler body confirmed no indirect consumer.
- `#2-apply-filter` `IN-REVIEW` developer -- Confirmed at HEAD: `inputType` appeared only in the param declaration. Implemented rather than removed: `niagara.search_modules` (`Source/PinWright/Private/Handlers/Niagara/NiagaraSearchHandler.cpp`) resolves `inputType` through `PinWrightNiagara::ResolveNiagaraParameterType` (`INVALID_PARAMETER_TYPE` otherwise), refuses it for any usage but `DynamicInput` (`INVALID_ARGUMENT`), and as the last filter loads each surviving candidate, reads the single input pin of its single output node (same shape `UNiagaraStackFunctionInput::GetAvailableDynamicInputs` uses) and keeps it when `FNiagaraEditorUtilities::AreTypesAssignable(output, requested)`. Matching rows now carry `outputs:[{name,type}]`; without `inputType` `outputs` stays empty and nothing is loaded. Tests (new file `Tests/Niagara/TestNiagaraSearchModulesInputType.cpp`): `PinWright.niagara.search_modules.InputTypeFiltersByOutputType` (engine `UniformRanged*` family: float and vector each narrow the unfiltered `totalMatches`, every float row's output type is `NiagaraFloat`-def name, every vector row Vector/Position, float and vector row sets disjoint; skip marker if the stock assets are absent) and `PinWright.niagara.search_modules.InputTypeRefusals`. Docs: `docs/wiki-src/niagara.md` (search_modules), param description, CHANGELOG. Behaviour change: `inputType` on a non-DynamicInput usage is now refused.
- `#3-verified-linux` `DONE` tester — Verified on Linux, UE 5.8, PinWright 7230b41d (commit b39ceafe). run3/full passed non-skipped: `PinWright.niagara.search_modules.InputTypeFiltersByOutputType` and `.InputTypeRefusals`. The stock-asset skip marker did not fire, and the test is not in skips-all. Acceptance, taking the filter-implemented branch: on the engine `UniformRanged*` DynamicInput family, float and vector each narrow the unfiltered `totalMatches`. Every float row's output type is Float, every vector row's is Vector or Position, and the two row sets are disjoint. So two types give different result sets, as the ticket's regression asked. Matching rows carry `outputs:[{name,type}]`. An unknown type is refused `INVALID_PARAMETER_TYPE`, and `inputType` on a non-DynamicInput usage is refused `INVALID_ARGUMENT`. That second refusal is a behaviour change, recorded in CHANGELOG.
