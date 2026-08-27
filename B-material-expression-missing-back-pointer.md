---
id: B-material-expression-missing-back-pointer
title: "PinWright-created material expressions have a null `Material` back-pointer, so expression-level `property.set` cannot invalidate the material"
status: IN-REVIEW
severity: High
category: bug
tags: [material, property-set, silent-no-effect]
---

# PinWright-created material expressions have a null `Material` back-pointer

Reproduced live, UE 5.8, plugin `d195a55d`:

```
material.authoring.create_material  name="M_PwProbe"
material.authoring.add_custom_expression code="return float3(1,0,0);" outputType="Float3"
  -> nodeId 73ED484A4E33077AAF179095BFB25154

property.get objectPath=".../M_PwProbe.M_PwProbe:MaterialExpressionCustom_0" propertyName="Material"
  -> {"value": null}
```

`UMaterialExpression::Material` is the back-pointer the engine gates its own change-forwarding on.
With it null, an expression-level edit cannot reach the material.

## Mechanism

`property.set` does call `RootObject->PostEditChange()` (`Handlers/Utility/UtilityPropertyHandler.cpp:1129`)
— that part is fine and has been there since the squashed-history root. The forward from the *expression*
to the *material* is what fails.

`Runtime/Engine/Private/Materials/MaterialExpressions.cpp:1587-1603`
(`UMaterialExpression::PostEditChangeProperty`, reached from `UMaterialExpressionCustom::PostEditChangeProperty`
at `:12842-12871`):

```cpp
if (Material && !(Material->bIsPreviewMaterial || Material->bIsFunctionPreviewMaterial))
{ Material->PreEditChange(nullptr); Material->PostEditChangeProperty(SubPropertyChangedEvent); }
```

`Material` is a plain serialized `UPROPERTY` (`Runtime/Engine/Public/Materials/MaterialExpression.h:183-184`).
PinWright never assigns it: `Handlers/Material/MaterialAuthoringHandler.cpp:1560-1590` creates the node with
`NewObject<UMaterialExpressionCustom>(Material, ...)` and sets `Code`, `Inputs`, `OutputType` and the editor
position — no `CustomExpr->Material = Material;`. Same omission in
`Material/MaterialExpressionFactory.cpp:253-261`. A repo-wide grep for `->Material =` across the material
handlers returns only an unrelated resolve-result struct.

The engine's own paths do set it — `Editor/MaterialEditor/Private/MaterialEditingLibrary.cpp:655` and `:699`
(`NewExpression->Material = Material;`) — and `MaterialEditor.cpp:250-265`
`RestoreExpressionBackReferences` exists precisely to repair it, with the comment *"Restore back-pointers
that serializers and duplicators null out."* That runs only when the Material Editor opens the asset, which
is why a hand-opened material behaves differently from an MCP-created one.

## Fix

One line at each create site: `NewExpression->Material = Material;`, matching `MaterialEditingLibrary`.

## Note on the original report

This ticket deliberately does **not** claim that `compile_material` reuses a stale shader. That claim does
not hold: `compile_material` runs `PreEditChange(nullptr)` + `PostEditChange()`, reaching
`UMaterial::PostEditChangePropertyInternal` (`Runtime/Engine/Private/Materials/Material.cpp:5350`) with a null
`Property`, so `bRequiresCompilation` stays true and `:5412-5418` runs `UpdateCachedExpressionData()` +
`CacheResourceShadersForRendering(bRegenerateId=true)` — a full retranslate. Today it goes further, through
`ApplyMasterMaterialEdit` (`MaterialAuthoringHandler.cpp:3512`) wrapping the pair in an
`FMaterialUpdateContext` so dependent instances recache. The real defect is only the missing back-pointer.

## Related hazard, same surface

`property.set` has no `EDITOR_OPEN` guard. The material namespace refuses graph mutation while
`FMaterialEditor` is open (`Handlers/Material/MaterialFinders.h:84`, `:142-144`, `:260-263`) because the
editor edits a working copy and clobbers external mutations on its next sync — documented at
`wiki-src/material.authoring.md:54`. `property.set` bypasses that guard entirely, so editing `Code` through
it with the Material Editor open silently loses the edit. Worth folding into the same fix or a sibling ticket.

## Coverage gap

`Tests/Assets/TestMaterialHandlers.cpp:540-660` covers `add_custom_expression` inputs and nodeId only.
`Tests/Material/TestCompileMaterialShaderErrors.cpp:54-62` sets `Custom->Code` directly in C++, never through
`property.set`. No test asserts that `UMaterialExpression::Material` is non-null after a PinWright create —
which is a one-line assertion that would have caught this at every create site.

## History

- `#1-reported-with-live-repro` `OPEN` reporter — Confirmed at runtime on `d195a55d` / UE 5.8: `property.get` on a freshly created `MaterialExpressionCustom` returns `Material: null`. Engine gate and the engine's own assigning call sites verified in source. The stale-shader half of the original report was checked and does not hold; recorded above so it is not re-filed.
- `#2-back-pointer-set-at-every-create-site` `IN-REVIEW` developer — Every production expression-create site now assigns the owning back-pointer. `Material/MaterialExpressionFactory.cpp` (the shared path behind ~30 typed `add_*` verbs, `material.graph.*` and the MGIR emitter): `NewExpr->Material = Material` in the `UMaterial` overload, `NewExpr->Function = Function` in the `UMaterialFunction` overload (Material stays null there — that is the pair `UMaterialExpression::PostEditChangeProperty`'s `else if (Function)` branch needs, and the pair the material editor writes back when it saves a function graph). `Handlers/Material/MaterialAuthoringHandler.cpp`: `add_custom_expression` (the ticket's repro), `add_function_input`, `add_function_output`, and the `FINALIZE_EXPR_AND_RESPOND` macro used by `use_material_function`. Also the two same-defect sites outside the ticket's citations: `Handlers/Material/MaterialGraphHandler.cpp` (`material.graph.add_texture_sample`) and the `MPC_FINALIZE_EXPR_AND_RESPOND` macro in `Handlers/Material/MaterialParameterCollectionHandler.cpp`. New test file `Tests/Material/TestMaterialExpressionOwnerBackPointer.cpp` adds `PinWright.material.authoring.expression_back_pointer.CreatedExpressionsResolveOwningMaterial` (drives four distinct creation paths against an empty fixture material, then sweeps the whole expression collection for orphans) and `...CreatedExpressionsResolveOwningFunction` (function graph: `Function` set, `Material` null). No repair-on-read migration was added — see report; it is a loop, not a one-liner, and the natural place for it is `property.set`'s object resolution, not the material namespace.
