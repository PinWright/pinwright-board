---
id: B-material-main-output-pins-incomplete
title: "connect_nodes / break_connections to main material node miss several output inputs"
status: DONE
severity: High
category: bug
tags: [material, material-graph, material-authoring, main-node, connect-nodes, coverage-gap]
---

# Main material node wiring covers only a subset of inputs

Both `material.authoring.connect_nodes` and `material.graph.connect_nodes`
route into the main material node via an inlined `if/else if` ladder over
hard-coded input names. Same for `break_connections` and
`disconnect_nodes`. The supported set is:

`BaseColor, EmissiveColor, Roughness, Metallic, Specular, Normal,
Opacity, OpacityMask, AmbientOcclusion, SubsurfaceColor` — plus
`WorldPositionOffset` in `material.authoring.connect_nodes` only,
NOT in `material.graph.connect_nodes`.

Missing engine inputs on `UMaterialEditorOnlyData` / `FMaterialInput` that
ship in UE 5.6 stock surface / decal / post-process / UI materials:

- `Refraction`
- `PixelDepthOffset`
- `Anisotropy`
- `Tangent`
- `ClearCoat`
- `ClearCoatRoughness`
- `Displacement` (renamed from `WorldDisplacement` in 5.4)
- `ShadingModelFromMaterialExpression`
- `SurfaceThickness`
- `CustomizedUVs[0..7]` (eight indexed array slots)
- `WorldPositionOffset` (missing from `material.graph` only)
- `FrontMaterial` (Substrate root input — used by every Substrate-enabled
  material; not mentioned in the original report but in the same code path)

The `MaterialAttributes` input on materials configured with
`UseMaterialAttributes` is also missing — that's the canonical wiring
used by layered materials.

**Why it matters:**

- Decals can't be wired (need `Roughness/Metallic/Normal/Opacity` which
  work, but advanced surface decals want `EmissiveColor + Opacity` which
  also work — minor). However post-process materials need `EmissiveColor`
  (works) and `Opacity` (works) but advanced shading models like
  `ClearCoat` and `Anisotropy` are blocked.
- Vertex animation pipelines need `WorldPositionOffset` (works through
  authoring but NOT graph) and `Displacement` (blocked).
- Refraction blocked entirely.
- Customized UVs (used to drive per-pixel UV manipulation through the
  material output stage) — blocked entirely, eight slots missing.
- Layered/MaterialAttributes materials can't be authored at all.

The ladder repetition between `MaterialAuthoringHandler.cpp` (~L1278) and
`MaterialGraphHandler.cpp` (~L221) is also a maintenance trap — adding
a new input requires editing two places.

**Fix:**

Replace the duplicated `if/else if` chain with a table-driven helper:

```cpp
struct FMainInputBinding {
    const TCHAR* Name;
    TFunction<FExpressionInput*(UMaterialEditorOnlyData*)> Get;
};
static const FMainInputBinding GMainInputs[] = {
    {TEXT("BaseColor"), [](auto* D){ return &D->BaseColor; }},
    {TEXT("EmissiveColor"), [](auto* D){ return &D->EmissiveColor; }},
    {TEXT("Roughness"), [](auto* D){ return &D->Roughness; }},
    /* … all of the above … */
};
```

Drive `connect_nodes` / `break_connections` from this table. For
`CustomizedUVs`, accept `CustomizedUVs[0]` … `CustomizedUVs[7]` (index
into `D->CustomizedUVs`).

For materials with `bUseMaterialAttributes`, route `MaterialAttributes`
into `D->MaterialAttributes`.

When the input name doesn't match the table, surface a structured error
listing the valid names for the current material domain
(`Surface`/`PostProcess`/`UI`/`Decal`/...) — domain narrows the valid
set.

## Repro

1. Create material `M`, `set_material_domain("Surface")`,
   `set_shading_model("ClearCoat")`.
2. `add_scalar_parameter(name: "CC")` → nodeId `N`.
3. `material.graph.connect_nodes(materialPath: M, sourceNodeId: N,
   targetNodeId: "", inputName: "ClearCoat")` → `INVALID_PIN: Unknown
   input on main node: ClearCoat`. Same error for `Refraction`,
   `Anisotropy`, `WorldPositionOffset` (graph-only), `Displacement`,
   `PixelDepthOffset`, `CustomizedUVs[0]`.

## History
- `#1-main-output-coverage-gap` `OPEN` reporter — Material API audit found `material.authoring.connect_nodes` and `material.graph.connect_nodes` only support 10–11 main-node inputs (BaseColor / EmissiveColor / Roughness / Metallic / Specular / Normal / Opacity / OpacityMask / AmbientOcclusion / SubsurfaceColor, plus WorldPositionOffset on authoring only). Missing: Refraction, PixelDepthOffset, Anisotropy, Tangent, ClearCoat, ClearCoatRoughness, Displacement, ShadingModelFromMaterialExpression, SurfaceThickness, CustomizedUVs[0..7], MaterialAttributes, and WorldPositionOffset on the graph variant. The same `if/else if` ladder is duplicated across two handlers. Fix is a single static table mapping input name → `&UMaterialEditorOnlyData::Field` that both handlers iterate; also handles `break_connections` / `disconnect_nodes` parity.
- `#2-reviewed-and-confirmed` `OPEN` tester — Verified against UE 5.6 sources. `UMaterialEditorOnlyData` in `Engine/Public/Materials/Material.h` declares every input named in the report (lines 316–393) plus `FrontMaterial` (Substrate root, line 396) and the 8-slot `CustomizedUVs[8]` array (line 381). Confirmed four duplicate ladders: `MaterialGraphHandler.cpp` :221 (connect, 10 inputs) and :314 (disconnect, 10), `MaterialAuthoringHandler.cpp` :1278 (connect, 11 with WPO) and :1362 (disconnect). No duplicate board entry. Severity `High` justified — Substrate materials, Refraction, ClearCoat, Displacement, layered (`MaterialAttributes`) and any UV-customization workflow are entirely blocked from end-to-end MCP authoring. Note on fix shape: `FExpressionInputIterator` walks inputs on a `UMaterialExpression`, but the main-material node is `UMaterial` itself (inputs live on `UMaterialEditorOnlyData`), so the iterator pattern from `MaterialExpression.h:660` does *not* apply here. A hand-curated `{Name, FExpressionInput*(UMaterialEditorOnlyData*)}` table — as the report proposes — is the correct approach; CustomizedUVs needs index-suffix parsing (`CustomizedUVs[N]` → `&D->CustomizedUVs[N]` with bounds check). Reflection over `UMaterialEditorOnlyData::StaticClass()` properties is a future-proofing alternative but adds complexity for a stable schema. `FrontMaterial` should be added to the planned table.
- `#3-table-driven-main-bindings` `IN-REVIEW` developer — Added Handlers/Material/MainInputBindings.h with a 22-entry static table mapping main-node input names to UMaterialEditorOnlyData fields plus CustomizedUVs[N] index parsing. Replaced 4 if/else ladders in MaterialGraphHandler.cpp (connect/disconnect main branches) and MaterialAuthoringHandler.cpp (connect/disconnect main branches). MaterialAuthoringHandler disconnect silent-success on unmatched main-pin (~L1369) now SendError(INVALID_PIN). Domain-narrowed error list deferred. Regression test TestMaterialMainNodeInputCoverage.cpp asserts ClearCoat/Refraction/Anisotropy/PixelDepthOffset/Displacement/WorldPositionOffset/CustomizedUVs[0]/[7] wire correctly and CustomizedUVs[8] rejects.
- `#4-verify-clearcoat-main-pin` `DONE` tester — Verified: created `/Game/McpVerify/M_McpVerifyTemp_B_material_main_output_pins_incomplete`, added scalar parameter node `DDE026484294ECA60AB001856BEEA50B`, and `material.graph.connect_nodes` with `targetNodeId: ""`, `inputName: "ClearCoat"` returned `inputName: "ClearCoat"` instead of `INVALID_PIN`; cleanup deleted the temp material with `asset.delete path`.
