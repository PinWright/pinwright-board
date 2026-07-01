---
id: F-material-layers-asset-authoring
title: "No RPC to create MaterialFunctionMaterialLayer / LayerBlend assets or author layer stacks"
status: DONE
severity: Medium
category: feature
tags: [material, material-authoring, material-layers, mgir, coverage-gap]
---

# Material layer / blend assets cannot be created via RPC

UE's material-layers system is built on two specialised
`UMaterialFunctionInterface` subclasses:

- `UMaterialFunctionMaterialLayer` — a layer (one set of material
  attribute outputs).
- `UMaterialFunctionMaterialLayerBlend` — a blend (combines two layers
  into one).

These are normal content assets, authored in the editor via the
*Material Function (Layer)* / *Material Function (Material Layer Blend)*
factory entries. They are referenced from materials via
`UMaterialExpressionMaterialAttributeLayers`, which holds parallel
`Layers[]` / `Blends[]` arrays of these function-interface pointers.

## What exists today

- `material.authoring.create_material_function` (MaterialAuthoringHandler.cpp:1337)
  creates a plain `UMaterialFunction` via `UMaterialFunctionFactoryNew`.
  This is **not** a layer or a blend — the resulting asset cannot be
  assigned to `LayersExpression->DefaultLayers.Layers` or `.Blends`.
- `MGIRExpressionEmitter.cpp:386–427` can wire a
  `UMaterialExpressionMaterialAttributeLayers` node and populate its
  `DefaultLayers.Layers` / `.Blends` arrays from existing
  `UMaterialFunctionInterface` references in the MGIR symbol table.
  The emitter assumes the layer and blend assets **already exist**.

## What is missing

There is no RPC that creates the underlying layer or blend asset.
Today an agent has to either:

1. Manually create the assets in the editor before invoking MGIR
   (defeats automation), or
2. Reuse engine-provided defaults like
   `/Engine/Functions/MaterialLayerFunctions/MLB_Standard` — fine for
   stock blends, useless for any custom layer/blend authoring.

The layer/blend mutation done inline by MGIR
(`LayersExpression->DefaultLayers.Layers.Add(...)` + parallel
`EditorOnly.LayerNames` / `LayerStates` / `LayerGuids` /
`LayerLinkStates` / `RestrictToLayerRelatives` array maintenance) also
isn't exposed as a typed standalone handler, so callers driving a
non-MGIR material edit path can't reproduce it.

## Proposed RPCs

```
material.authoring.create_material_layer
    name, path? (default /Game/Materials/Layers), description?,
    exposeToLibrary? (default true), save? (default true)
    → { assetPath, className }

material.authoring.create_material_layer_blend
    name, path? (default /Game/Materials/LayerBlends), description?,
    exposeToLibrary? (default true), save? (default true)
    → { assetPath, className }

material.authoring.set_material_layer_stack
    assetPath, expressionId,
    layers: [assetPath, ...],
    blends: [assetPath, ...],  // length must be layers.length - 1
    save? (default true)
    → { layersApplied, blendsApplied }
```

Implementation notes:

- Layer/blend creation uses
  `UMaterialFunctionMaterialLayerFactory` and
  `UMaterialFunctionMaterialLayerBlendFactory` from
  `Editor/UnrealEd/Classes/Factories/` — same shape as
  `UMaterialFunctionFactoryNew` already used by
  `create_material_function`.
- `set_material_layer_stack` should encapsulate the parallel-array
  maintenance currently inline in
  `MGIRExpressionEmitter::EmitMaterialAttributeLayers` (LayerNames,
  LayerStates, LayerGuids, LayerLinkStates, RestrictToLayerRelatives,
  RestrictToBlendRelatives) and call `RebuildLayerGraph(false)`. MGIR
  can then delegate to it instead of duplicating the logic.
- Class check: factories return
  `UMaterialFunctionMaterialLayer` / `UMaterialFunctionMaterialLayerBlend`
  which derive from `UMaterialFunctionInterface`. The existing
  `material.authoring.create_material_function` returns the concrete
  `UMaterialFunction` subclass and won't satisfy `Cast<...Layer>` /
  `Cast<...LayerBlend>` checks performed when assigning to the
  expression's `Layers[]` / `Blends[]` arrays.

## Why it matters

Material layers are how UE artists compose surface materials from
reusable substance-style stacks (e.g. *Base + Dirt + Wear*). Without
authoring RPCs an agent can only consume pre-built stock layers; it
cannot ship a project that uses custom layers/blends end-to-end.

## History
- `#1-layer-blend-assets-not-creatable` `OPEN` reporter — MGIR can populate `UMaterialExpressionMaterialAttributeLayers.DefaultLayers.Layers/.Blends` from existing `UMaterialFunctionInterface` references (MGIRExpressionEmitter.cpp:386–427) but no RPC creates the underlying `UMaterialFunctionMaterialLayer` / `UMaterialFunctionMaterialLayerBlend` assets. `material.authoring.create_material_function` returns the wrong concrete subclass (plain `UMaterialFunction`) and cannot be assigned to a layer/blend slot. Proposing `create_material_layer`, `create_material_layer_blend`, and a typed `set_material_layer_stack` that wraps the parallel-array mutation MGIR does inline today so non-MGIR callers can author layer stacks too.
- `#2-layer-blend-rpcs-and-shared-helper` `IN-REVIEW` developer — Added create_material_layer/create_material_layer_blend/set_material_layer_stack RPCs and a shared MaterialLayerStackHelpers.h ApplyMaterialLayerStack helper. Refactored MGIRExpressionEmitter::EmitLayerStack (parallel-array Adds + RebuildLayerGraph) to delegate to the helper for single source of truth. Regression tests assert Cast<UMaterialFunctionMaterialLayer> / Cast<UMaterialFunctionMaterialLayerBlend> succeed (the bug the original create_material_function couldn't satisfy).
- `#3-verify-fix` `DONE` tester — Verified: all three RPC schemas resolve via `material.authoring.create_material_layer?` / `create_material_layer_blend?` / `set_material_layer_stack?`. Live-created `/Game/App/UI/Test/MLayer_McpVerifyTemp_F-material-layers` and `MLBlend_...` via the new RPCs — responses returned `assetClass: "MaterialFunctionMaterialLayer"` and `"MaterialFunctionMaterialLayerBlend"` respectively (the exact concrete subclasses the original `create_material_function` couldn't produce). Temp assets deleted via `asset.delete`.
