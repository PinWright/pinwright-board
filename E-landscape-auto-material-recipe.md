---
id: E-landscape-auto-material-recipe
title: "No documented height/slope auto-material recipe: MGIR can build one but nothing says how, and configure_layer_blend's HeightBlend reads as if it were one"
status: OPEN
severity: Low
category: ergonomic
tags: [landscape, material, mgir, wiki, auto-material, gap-analysis-2026-09-30]
encounters: 1
---

# No documented landscape auto-material (height/slope) recipe

`material.authoring.create_landscape_material` builds an empty graph (`MaterialAuthoringHandler.cpp:3150`). `configure_layer_blend` (`:3365/:3403`) sets up painted Weight/Alpha/Height blending only. Nothing documents how to build a height or slope mask. pinwright.com/compare rates the row partial.

VibeUE's "yes" does not work. `SetupHeightSlopeBlend` (`ULandscapeMaterialService.cpp:1327`) sets the layer to LB_AlphaBlend and wires the mask into HeightInput. The engine only compiles HeightInput for non-AlphaBlend layers (`C:\UE_5.8\Engine\Source\Runtime\Landscape\Private\Materials\MaterialExpressionLandscapeLayerBlend.cpp:171-175`), and `PostEditChangeProperty` nulls HeightInput on every non-HeightBlend layer (`:352-358`). VibeUE's own SKILL.md:193 admits that auto-blend never appears when every layer weight is 0.

**Fix:** a `material.examples.landscape-auto` wiki page authored under `docs/wiki-src/`, following the `bpir.examples.*` pattern. It should contain:
- An MGIR document with a height mask (WorldPosition → ComponentMask B → SmoothStep), a slope mask (VertexNormalWS.z → OneMinus → SmoothStep, or `/Engine/Functions/Engine_MaterialFunctions01/AlphaBlend/WorldAlignedBlend` via `function_call`), and the chain `Lerp(Lerp(Base, Slope, slopeMask), Height, heightMask)`. Thresholds are ScalarParameters so instances can tune them. Optionally multiply by `LandscapeLayerSample` so painting can override.
- The follow-on calls: `landscape.set_material` → `material.authoring.compile_material` (read `consumerRefresh`) → a capture.
- A warning that LB_AlphaBlend and LB_HeightBlend ignore mask inputs.

Unverified risk: LayerBlend `Layers[]` holding `FExpressionInput` may not round-trip through MGIR (the only dynamic-input class is Custom, `MGIRDynamicInputs.cpp:10`). Keep the auto part in the Lerp chain and prove it with a fixture. Defer a `configure_auto_blend` verb until agents are seen failing with the recipe.

**Acceptance:**
- A fixture compiles the page's MGIR verbatim through `material.compile_mgir`.
- On a landscape with zero painted weights, a capture shows the slope layer on steep ground and the height layer above the threshold.

Effort S.

## History
- `#1-gap-analysis` `OPEN` reporter — Filed from the 2026-09-30 competitor gap analysis (compare row "Landscape auto-material (height/slope blend)": PinWright partial, VibeUE yes). The capability exists through MGIR; the gap is that it is not discoverable. VibeUE's "yes" is non-functional per the engine source cited above.
