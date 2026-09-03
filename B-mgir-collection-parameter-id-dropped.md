---
id: B-mgir-collection-parameter-id-dropped
title: "MGIR drops UMaterialExpressionCollectionParameter.ParameterId — decompile omits the FGuid and compile never resolves it from ParameterName, so any material with an MPC node fails to compile after a round trip, silently unless the node sits on a live branch"
status: OPEN
severity: High
category: bug
tags: [mgir, compile_mgir, decompile_mgir, material-parameter-collection, collection-parameter, parameterid, fguid, round-trip, data-loss, silent-failure]
encounters: 1
lastSeen: 2026-09-03T07:20:00+03:00
---

# MGIR creates `CollectionParameter` nodes with a null `ParameterId`, so the material fails to compile

`UMaterialExpressionCollectionParameter::Compile` resolves the parameter **by
`ParameterId` (an `FGuid`), never by `ParameterName`**
(`Runtime/Engine/Private/Materials/MaterialExpressions.cpp:17179-17204`):

```cpp
if (Collection) { Collection->GetParameterIndex(ParameterId, ParameterIndex, ComponentIndex); }
if (ParameterIndex != -1) { return Compiler->AccessCollectionParameter(...); }
else { ... return Compiler->Errorf(TEXT("CollectionParameter has invalid parameter %s"), *ParameterName.ToString()); }
```

`material.decompile_mgir` **does not emit `ParameterId`**, and
`material.compile_mgir` **does not resolve it**. A node created from MGIR text
therefore carries `Collection` + `ParameterName` and a **zero-value `FGuid`**,
and the material fails translation.

## Repro (live, UE 5.8, port 27145, 2026-09-03)

```
material.authoring.create_parameter_collection  name=MPC_WPN_Viewmodel path=/Game/FPS/Weapons/Materials
material.authoring.add_collection_scalar_parameter  assetPath=.../MPC_WPN_Viewmodel name=ViewmodelFOVScale default=1.0
  -> {"added":true,"parameterId":"2EBBE2E64562E9562B8B95B029697A69"}

# typed verb binds correctly:
material.authoring.add_collection_parameter_node  assetPath=<mat> collectionPath=.../MPC_WPN_Viewmodel parameterName=ViewmodelFOVScale x=320 y=3060
  -> {"nodeId":"B23EA2574B13205176CA10BE8790AB05"}

# decompile that same node -- ParameterId is absent:
material.decompile_mgir  assetPath=<mat>
  -> %nB23EA2574B13 = call `/Script/Engine.MaterialExpressionCollectionParameter`(
         Desc: "...", Collection: "/Game/.../MPC_WPN_Viewmodel.MPC_WPN_Viewmodel",
         ParameterName: "ViewmodelFOVScale") @(320, 3060)

# recompiling that emitted text produces an unbound node:
material.compile_mgir  mode=Append text=<the decompiled document>
material.authoring.compile_material  assetPath=<mat>
  -> {"compileSucceeded":false,"compileStatus":"failed","compiledWithErrors":true,
      "compileErrors":["(Node CollectionParameter) CollectionParameter has invalid parameter ViewmodelFOVScale"]}
```

So the documented **decompile -> edit -> `Append` round trip** — the mode
`material.mgir` explicitly recommends for editing an existing graph ("`Append`
with the full document ... which is the mode round-trip editing is for") —
**destroys every MPC node in the graph**.

## Why it is easy to ship broken

The failure is invisible whenever the collection node feeds a branch that is not
translated. In the case that surfaced this, the node fed the `True` input of a
`StaticSwitchParameter` whose `DefaultValue` is `false`. `compile_material`
reported `compileSucceeded: true, compileErrors: []` because the engine compiles
only the taken branch — the broken node was never visited. The asset saved
clean, and the defect would have detonated only when a material instance set the
switch to `true`. It was caught only by deliberately flipping the switch default
to `true`, compiling, and flipping it back.

`asset.dump` writes the same decompiler output to `mgir.txt`, so pinned MGIR
baselines for any material containing an MPC node are already lossy on disk.

## Mechanism

`ParameterId` is a bare `UPROPERTY()` with no `EditAnywhere`
(`Runtime/Engine/Public/Materials/MaterialExpressionCollectionParameter.h`), so
it is filtered out of the decompiler's reflected-property emit even though
`FStructProperty` is otherwise accepted by `IsSafeReflectedProperty` (per
`B-mgir-tarray-properties-dropped`). Nothing on the compile side compensates.

Note `PostLoad` makes it worse rather than better
(`MaterialExpressions.cpp:17151-17160`): it recomputes
`ParameterName = Collection->GetParameterName(ParameterId)`. With a null
`ParameterId` that resolves to `None`, so a saved-then-reloaded asset loses the
**name** as well, and the node can no longer be repaired by name at all.

## Workaround (verified)

`PostEditChangeProperty` resolves the GUID from the name
(`MaterialExpressions.cpp:17163-17177`: `ParameterId = Collection->GetParameterId(ParameterName)`),
and `property.set` runs that notification. After the MGIR compile, re-set
`ParameterName` to the value it already has:

```
property.set  objectPath="/Game/.../M_X.M_X:MaterialExpressionCollectionParameter_1"
              propertyName="ParameterName"  value="ViewmodelFOVScale"
```

then `material.authoring.compile_material` -> `compileSucceeded: true`. The
expression's object name must be discovered first (it is not returned by
`compile_mgir`, and `material.graph.get_node_details` does not report
`ParameterId` either — see `B-material-get-node-details-missing-pins-props`);
`python.execute` with `unreal.find_object(material, 'MaterialExpressionCollectionParameter_%d' % i)`
works. Note the index is not stable — an earlier orphaned node occupied `_0`,
so the live one was `_1`.

## Fix

Two-sided, matching the shape used for `B-mgir-tarray-properties-dropped`:

1. **Decompiler** — emit `ParameterId` for `UMaterialExpressionCollectionParameter`
   (whitelist the property explicitly rather than relying on edit-ability), so the
   binding round-trips.
2. **Compiler** — after setting `Collection` and `ParameterName` on a created
   `CollectionParameter`, resolve the GUID (`ParameterId = Collection->GetParameterId(ParameterName)`)
   or simply fire `PostEditChangeProperty`, so hand-written MGIR that names only the
   collection and parameter still produces a working node. This also makes the IR
   more usable by hand, since an author should not have to paste a GUID.

Additionally, `material.compile_mgir` should reject — not silently accept — a
`CollectionParameter` whose `Collection` is set but whose named parameter does
not exist on it, mirroring `add_collection_parameter_node`, which already
returns `INVALID_PARAMS` "rather than silently leaving an unbound node".

## History
- `#1-initial-repro` `OPEN` reporter — Hit while wiring a viewmodel-FOV WPO hook into `/Game/FPS/Weapons/Materials/M_WPN_Master` from an MPC scalar. `material.compile_mgir` (Append, 80 expressions) created the `CollectionParameter` node with `Collection` and `ParameterName` set and a null `ParameterId`; `compile_material` then failed with `CollectionParameter has invalid parameter ViewmodelFOVScale`. Confirmed the decompiler never emits `ParameterId` by round-tripping a node that `material.authoring.add_collection_parameter_node` had bound correctly. First compile of the same graph reported `compileSucceeded: true` **only because** the node sat on the untaken branch of a `StaticSwitchParameter` defaulting to `false` — the error appeared solely after flipping that default to `true`. Worked around with `property.set` on `ParameterName` to trigger `PostEditChangeProperty`, which resolves the GUID; verified `compileSucceeded: true` with the branch live, then restored the default.
