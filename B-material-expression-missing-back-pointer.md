---
id: B-material-expression-missing-back-pointer
title: "PinWright-created material expressions have a null `Material` back-pointer, so expression-level `property.set` cannot invalidate the material"
status: DONE
severity: High
category: bug
tags: [material, property-set, silent-no-effect]
encounters: 2
lastSeen: 2026-08-28T09:25:00+05:00
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

- `#3-verified-fixed-with-an-A-B-A-control` `DONE` verifier — 2026-08-28. Editor running the plugin built
  at `b79ba53e`; ticket repro re-run live, then the *mechanism* proven with a control rather than inferred
  from the pointer value.

  **The ticket's own repro is closed.** `create_material M_PwProbeBackPtr` (into `/Game/PinWrightScratch`)
  -> `add_custom_expression {code:"return float3(1,0,0);", outputType:"Float3", inputs:[]}` ->
  `property.get {propertyName:"Material"}` on `…M_PwProbeBackPtr:MaterialExpressionCustom_0` now returns
  `"value":"/Game/PinWrightScratch/M_PwProbeBackPtr.M_PwProbeBackPtr"` where `#1` measured `null`.

  **All six create sites `#2` claims, checked individually with `property.get`**, each returning the owning
  material path: `add_custom_expression` (`MaterialExpressionCustom_0`), `add_scalar_parameter` — the shared
  `MaterialExpressionFactory` path behind the ~30 typed `add_*` verbs (`MaterialExpressionScalarParameter_0`),
  `material.graph.add_texture_sample` (`MaterialExpressionTextureSample_0`), `use_material_function` /
  `FINALIZE_EXPR_AND_RESPOND` (`MaterialExpressionMaterialFunctionCall_0`), and
  `add_collection_parameter_node` / `MPC_FINALIZE_EXPR_AND_RESPOND` (`MaterialExpressionCollectionParameter_0`).
  The function-graph half behaves as `#2` describes and doubles as a shape control on `property.get` itself:
  on `MF_PwProbeBackPtr:MaterialExpressionFunctionInput_0` (`add_function_input`), `Function` reads
  `/Game/PinWrightScratch/MF_PwProbeBackPtr.MF_PwProbeBackPtr` while `Material` reads `null` — so the same
  verb and the same property name still emit `null` when the pointer really is null, and the non-null reads
  above are not an artifact of a changed response shape.

  **This is a behaviour change, not a better diagnostic.** Measured through the base material's cached
  expression data, which only refreshes when the expression forwards `PostEditChangeProperty` to the
  material — read via `get_material_instance_info` on a child `MI_PwProbeBackPtr`, whose `inherited`/
  `parameters` block is served from the parent's `CachedExpressionData`. A-B-A in one binary, one session,
  identical calls:
  - **A (back-pointer set):** baseline `inherited.scalar {"PwScalar":0.5}`. `property.set
    {objectPath:"…:MaterialExpressionScalarParameter_0", propertyName:"ParameterName",
    value:"PwScalarRenamed"}` -> instance reports `{"PwScalarRenamed":0.5}`, `parameters:[{name:
    "PwScalarRenamed"}]`. The forward fires.
  - **B (back-pointer nulled by `property.set {propertyName:"Material", value:null}` — the pre-fix state,
    reconstructed in the fixed binary):** the *identical* rename to `PwScalarOrphaned` returns
    `applied:true, markedDirty:true, value:"PwScalarOrphaned"`, and the instance still reports
    `{"PwScalarRenamed":0.5}`. That is the ticket's headline defect reproduced on demand: a write the verb
    calls applied, which the material never hears about.
  - **A' (back-pointer restored):** rename to `PwScalarRestored` -> instance reports
    `{"PwScalarRestored":0.5}`. Refresh returns.

  The only variable between B and A' is `UMaterialExpression::Material`, so the engine gate at
  `MaterialExpressions.cpp:1587-1603` is exactly what `#2`'s one-line assignments re-open, and the fix
  changes what happens to the material rather than what the response says about it.

  **Still open, deliberately not counted against this ticket:** the "Related hazard, same surface" above.
  `grep -n "EDITOR_OPEN\|IsAssetEditorOpen" Handlers/Utility/UtilityPropertyHandler.cpp` at `b79ba53e`
  returns no guard — only two unrelated comment hits about `UMaterialEditorOnlyData` — so `property.set`
  still bypasses the `FMaterialEditor`-open refusal that `material.authoring` enforces. `#2` never claimed
  it; the ticket text itself offers it as "the same fix or a sibling ticket". It wants its own ticket with
  its own repro (edit `Code` with the Material Editor open, confirm the edit is lost on the editor's next
  sync) and should not sit behind a closed back-pointer ticket.

  Probes left in place under `/Game/PinWrightScratch/`: `M_PwProbeBackPtr` (saved, 11708 bytes),
  `MI_PwProbeBackPtr`, `MF_PwProbeBackPtr`, `MPC_PwProbeBackPtr`. No shipping asset and no level touched.
