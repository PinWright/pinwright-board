---
id: B-material-abstract-expression
title: "Material graph creation accepts abstract expression classes and reaches UObject's abstract-allocation ensure"
status: IN-REVIEW
severity: High
category: bug
tags: [material, expression, abstract-class, ensure, validation, persistence]
---

# Class lineage is checked, instantiability is not

`FMaterialExpressionFactory::ResolveExpressionClass` accepts any class derived from
`UMaterialExpression` (`MaterialExpressionFactory.cpp:145-202`). Both material and function
factory paths then pass that caller-selected class directly to `NewObject<UMaterialExpression>`
(`:222-254` and `:315-347`) without rejecting `CLASS_Abstract`, `CLASS_Deprecated`, or
`CLASS_NewerVersionExists`.

The base `/Script/Engine.MaterialExpression` is itself declared `UCLASS(abstract, ...)` in UE 5.8.
Creating it reaches `StaticAllocateObjectErrorTests`, which logs that the abstract instance will be
nulled on save and executes `ensureMsgf(false)` (`UObjectGlobals.cpp:3267-3296`). Thus
`material.graph.add_expression` or `add_node` can turn a valid class reference into an editor assert
and, if execution continues, an expression that cannot persist correctly.

The Niagara create-node path in this tree already contains the exact fix shape:
`ValidateCreateNodeClass` rejects all three class flags before constructing its checked factory.

## What it should do

Add the same class-flag gate to `FMaterialExpressionFactory::Create` before property validation or
`NewObject`, return a typed `CLASS_NOT_INSTANTIABLE` error, and cover both material and function
outers.

## Workaround

Use the default `includeAbstract:false` discovery result and never pass an abstract or deprecated
class name directly.

## Related

- `F-search-api-material-expressions`

## Fix

The factory previously performed lineage validation but passed flagged expression classes into property reflection and `NewObject`; the fix now rejects `CLASS_Abstract`, `CLASS_Deprecated`, and `CLASS_NewerVersionExists` first with the typed `CLASS_NOT_INSTANTIABLE` error. Both material and material-function paths share this guard, and the handler verbs inherit it without changing `ResolveExpressionClass` discovery behavior.

Files changed:
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Material\MaterialExpressionFactory.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\ErrorCodes.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Material\TestMaterialExpressionFactorySafety.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Material\MaterialTestHelpers.h` (shared fixture/count helpers)
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Docs\wiki-src\material.graph.md`

Exact tests:
- `PinWright.material.graph.factory.RejectsNonInstantiableExpressionClasses`
- `PinWright.material.graph.factory.HandlersRejectNonInstantiableExpressionClasses`

Deliberate non-changes: flagged classes remain resolvable for discovery; no truly abstract base is allocated; `create_nodes` retains its per-node `failCount` batch contract; unrelated MRQ edits, engine source, builds, live PIE, and commits were untouched.

## History
- `#1-source-scan` `OPEN` reporter -- Concrete input is the abstract base class itself; engine
  failure behavior and persistence warning were verified from UE 5.8 source without running it.
- `#2-class-instantiation-guard` `IN-REVIEW` developer -- Added pre-allocation class-flag rejection and structural factory/handler coverage; static verification only.
