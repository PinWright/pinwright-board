---
id: B-material-abstract-expression
title: "Material graph creation accepts abstract expression classes and reaches UObject's abstract-allocation ensure"
status: OPEN
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

## History
- `#1-source-scan` `OPEN` reporter -- Concrete input is the abstract base class itself; engine
  failure behavior and persistence warning were verified from UE 5.8 source without running it.
