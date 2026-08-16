---
id: B-configure-layer-blend-wrong-nodes
title: "material.authoring.configure_layer_blend creates ScalarParameter nodes, not a LandscapeLayerBlend — it does not make a landscape paintable despite its name and its add_landscape_layer cross-reference"
status: IN-REVIEW
severity: High
category: bug
tags: [material-authoring, landscape, layer-blend, silent-wrong-node, misleading-success, docs]
encounters: 1
lastSeen: 2026-08-13T05:57:34Z
---

# `configure_layer_blend` builds scalar parameters instead of a `LandscapeLayerBlend`

`material.authoring.configure_layer_blend` reports success and returns a `layerCount` plus
`nodeIds[]` — but the nodes it created are `UMaterialExpressionScalarParameter`s **named
after the requested layers**, not a `UMaterialExpressionLandscapeLayerBlend` (nor
`LandscapeLayerWeight` nodes). Nothing samples those scalars, they declare no landscape
target layers, and they are not connected to anything.

So the verb does **not** make a landscape paintable, despite its name, despite its
registered summary ("Configure landscape layer blend by adding weight parameters for each
layer"), and despite the sibling verb `material.authoring.add_landscape_layer` cross-referencing
it as the way to do so — its registered summary at
`MaterialAuthoringHandler.cpp:2503` still reads *"Create a ULandscapeLayerInfoObject (one of
the per-layer assets a landscape material binds to). **Used by configure_layer_blend to set
up weight-blended layers.**"* A caller who follows that chain
(`create_landscape_material` -> `add_landscape_layer` xN -> `configure_layer_blend` ->
`compile_material`) gets a clean success at every step and a landscape material with **zero**
target layers.

## Why this matters beyond the verb

This is the strong candidate for the **upstream cause** of the
`B-create-procedural-terrain-paints-nothing` repro. That ticket's root cause is "the
landscape's material had no `LandscapeLayerBlend` node and zero target layers —
`target_layers` held only `__LANDSCAPE_VISIBILITY__`". The caller had used
`configure_layer_blend`, believed the landscape was now paintable, and it never was. The
paint verb's false success was the *second* silent failure in the chain; this is the first.

## Source (verified 2026-08-13 at HEAD)

`Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:2587-2662`. The
per-layer loop (`:2632-2643`):

```cpp
UMaterialExpressionScalarParameter* WeightParam =
    NewObject<UMaterialExpressionScalarParameter>(Material, ..., RF_Transactional);
WeightParam->ParameterName = FName(*LayerName);
WeightParam->DefaultValue = (i == 0) ? 1.0f : 0.0f;
...
Material->GetEditorOnlyData()->ExpressionCollection.Expressions.Add(WeightParam);
```

No `UMaterialExpressionLandscapeLayerBlend` is ever constructed, no `Layers` array is
populated, and no output is wired into the material's base-colour (or any) input. The
`blendType` the caller passes per layer is read only far enough to satisfy the loop — it does
not select a `ELandscapeLayerBlendType` on anything (see
`E-material-configure-layer-blend-blendtype-undiscoverable`, which is the *discoverability*
angle on that same parameter and is therefore mostly moot until this is fixed).

## Where this is currently recorded

Only inside the body of `B-create-procedural-terrain-paints-nothing` (section "Related, found
while fixing (separate tickets needed)") and as a one-line caveat added to
`docs/wiki-src/landscape.md:28`. It has no ticket of its own, so it is invisible to the fix
picker. That is what this entry corrects; the detail stays in place in both locations.

**Workaround:** build the layer blend yourself from `python.execute` via
`MaterialEditingLibrary` — create a `MaterialExpressionLandscapeLayerBlend`, fill its `Layers`
array with `FLayerBlendInput` entries (`LayerName`, `BlendType`, `PreviewWeight`), connect its
output, then recompile. Confirm afterwards by reading the landscape's target layers, never by
trusting `configure_layer_blend`'s success.

**Fix:** two parts, either of which alone leaves a trap.
1. Make the verb do what its name says: create/reuse a
   `UMaterialExpressionLandscapeLayerBlend`, populate `Layers` from the request
   (`name` -> `LayerName`, `blendType` -> `LB_WeightBlend` / `LB_AlphaBlend`), connect its
   output to the requested material input (default base colour), and **read back the
   resulting target layers** in the response
   (`ALandscapeProxy::RetrieveTargetLayerNamesFromMaterials` on 5.5+ / `GetLayersFromMaterial`
   on 5.3-5.4 — the accessor `B-create-procedural-terrain-paints-nothing`'s fix already
   adopted) rather than echoing a `layerCount` that only counts nodes it made.
2. Correct `add_landscape_layer`'s registered summary
   (`MaterialAuthoringHandler.cpp:2503`) so it stops steering callers here as the
   make-it-paintable step, and keep the `docs/wiki-src/material.authoring.md` overlay honest
   about what the verb produces.

If (1) is judged out of scope, the verb should at minimum be renamed or made to fail loudly —
silently producing disconnected scalar parameters under a landscape-blend name is worse than
having no verb.

severity rationale: impact=silent-false-success producing a wrong graph (the caller is told
the landscape blend is configured; the material has zero target layers) x reach=the documented
landscape-material authoring path, and it is the upstream half of an already-High
paints-nothing ticket -> High.

## Related

- `B-create-procedural-terrain-paints-nothing` — the downstream false success; its body
  records this defect and its wiki fix already carries the caveat. Cross-referenced both ways.
- `E-material-configure-layer-blend-blendtype-undiscoverable` — `blendType` value-space
  discoverability on this same parameter; largely moot until the node type is right, and its
  proposed doc text must be written against the fixed behaviour, not the current one.
- `B-landscape-set-material-stale-mics` — the other verb in the same recovery chain; even a
  correctly-built layer blend will not render until that one is fixed.

## History
- `#1-split-from-create-procedural-terrain` `OPEN` reporter — "`material.authoring.configure_layer_blend` creates `UMaterialExpressionScalarParameter` nodes named after the requested layers instead of a `UMaterialExpressionLandscapeLayerBlend`, so it does NOT make a landscape paintable despite its name and despite `add_landscape_layer` cross-referencing it as the way to do so. Source-verified at HEAD: `MaterialAuthoringHandler.cpp:2632-2643` constructs only ScalarParameters, adds them to `ExpressionCollection.Expressions`, wires nothing, and populates no `Layers` array; the response echoes a `layerCount` that counts the nodes it made, not target layers the landscape can paint into. `add_landscape_layer`'s registered summary (`:2503`) still says 'Used by configure_layer_blend to set up weight-blended layers', so the whole documented chain reports success and yields a material with zero target layers. Strong candidate for the upstream cause of the `B-create-procedural-terrain-paints-nothing` repro (that landscape's material had no LandscapeLayerBlend and only `__LANDSCAPE_VISIBILITY__`): the caller used this verb, believed the landscape was paintable, and it never was. Previously recorded only inside that ticket's body and as a caveat at `docs/wiki-src/landscape.md:28`; split out here so the fix picker can see it, with the detail left in place in both locations. Fix: build a real LandscapeLayerBlend, populate Layers from name/blendType, connect the output, and read the resulting target layers back via `RetrieveTargetLayerNamesFromMaterials` (5.5+) / `GetLayersFromMaterial` (5.3-5.4); also correct `add_landscape_layer`'s summary."
- `#2-node-type-fixed` `IN-REVIEW` developer — First half fixed in `1aa2b4b3`. The verb now writes the requested names into a `UMaterialExpressionLandscapeLayerBlend` (`MaterialAuthoringHandler.cpp:2749-2788` finds or creates the material's first blend node; `:2800-2824` populates `Layers` from `name`/`blendType` and carries over the `LayerInput`/`HeightInput` wiring of any layer name that survives the rewrite; `:2826-2867` handles connection). `MinimalAPI` still exports `Z_Construct_UClass_UMaterialExpressionLandscapeLayerBlend` as `LANDSCAPE_API` and Landscape is a public dependency (`PinWright.Build.cs:24`), so the class links directly with no reflection detour. Root cause confirmed as reported: a `ScalarParameter` declares no target layer because the engine harvests them by calling `GetLandscapeLayerNames` on every expression (`MaterialCachedData.cpp:566`) and the base implementation is empty — an audit found **zero** `MaterialExpressionLandscapeLayerBlend` constructions anywhere in the source tree. Verification now reads what the consumer reads: `targetLayers[]` comes from `UE::Landscape::RetrieveTargetLayerNamesFromMaterial` (`LandscapeUtils.cpp:577`), the accessor `ALandscapeProxy::RetrieveTargetLayerNamesFromMaterials` harvests through, **not** from the array just written, so a node that exists but declares nothing reports empty (`MaterialAuthoringHandler.cpp:2893-2900`). The three silent `continue`s named in `#1` became typed `INVALID_ARGUMENT` rejections naming the offending index — non-object (`:2700-2702`), nameless (`:2708-2710`), duplicate name (`:2723-2726`, which would otherwise declare fewer layers than `layerCount` reports because `GetLandscapeLayerNames` `AddUnique`s). `nodeIds` now carries the one configured blend node rather than one scalar parameter per layer; `blendNodeId` and `mode` (`created` / `updated_in_place`) are new, and extra blend nodes on the same material are left alone and counted in `otherLayerBlendNodes`. Explicitly distinguished from the `f92a4d32` fix, which wired this verb into `ApplyMasterMaterialEdit` so its edit *reaches* the landscape: pushing an edit and creating the right node are separate bugs, and only the first was addressed then — reporting them as one is how this got recorded complete while still broken end to end.
- `#3-wiring-fixed-and-both-cross-references-corrected` `IN-REVIEW` developer — Second half fixed in `1ae6021f`, superseding `1aa2b4b3`'s "Not fixed here" paragraph. Declaring target layers makes a layer paintable; it does not make the paint visible — an unconnected `LandscapeLayerBlend` compiles into no output, so the weights land in the weightmap and no shader samples them. That is the same "every step succeeded and the durable outcome is absent" shape as the node-type defect, one layer down, and this ticket named both halves, so leaving the wiring out was the wrong call. The verb now wires the blend node into a main material input via `connectTo` (default `BaseColor`) and **never overwrites**: an input already fed by another expression is left alone, `connectionState` reports `skipped_input_already_wired`, and `warnings[]` tells the caller the layers are paintable but unsampled and which verb fixes it (`MaterialAuthoringHandler.cpp:2834-2866`). `connectTo:"none"` opts out; an unknown input name is rejected with the valid list (`:2845-2848`). `connectionState` and `connectedTo` are deliberately separate response fields from `targetLayers` so "paintable" and "visible" cannot be read as one number. Both misleading registered summaries named in `#1` are corrected: `add_landscape_layer` (`:2518-2519`) now says a `ULandscapeLayerInfoObject` is bound per LANDSCAPE rather than by the material, that the paint verb auto-creates a private one when none exists, and points at `configure_layer_blend` as the thing that declares target layers; `create_landscape_material` (`:2401-2402`) now says its graph is EMPTY and declares nothing paintable, and points at the same verb.
- `#4-tests-assert-the-landscape-not-the-response` `IN-REVIEW` developer — Two regression tests, both chosen so the pre-existing coverage's blind spot cannot recur. `TestConfigureLayerBlendCreatesLayerBlendNode.cpp:57-59` asserts on the resulting MATERIAL with a stated differential property — pre-fix 0 `LandscapeLayerBlend` nodes / 3 `ScalarParameter`s / 0 declared names FAILS, post-fix 1 / 0 / 3 PASSES (`:22-32`). Its header records why the old coverage was worthless: `Tests/Assets/TestMaterialHandlers.cpp:1321-1323` (`configure_layer_blend.ValidParamsNoCrash`) asserts the response shape and **passes against the defect**, because a response that echoes the request back cannot fail. `TestConfigureLayerBlendLandscapeEndToEnd.cpp:104-106` walks the whole chain this ticket describes — `create_landscape_material` -> `configure_layer_blend` -> `landscape.create` -> paint — and asserts on the LANDSCAPE: `ALandscapeProxy::RetrieveTargetLayerNamesFromMaterials()` contains the requested names (`:216`) and the paint reports `texelsWithWeight > 0` (`:250-251`). Measured: `PinWright.material.authoring.configure_layer_blend` **4 performed / 4 pass / 0 fail**, from `Saved/Logs/pw_clb_v2.log` carrying `Automation Test Queue Empty 4 tests performed` with `started == success + fail`, against the binary linked at 13:08. The transferable rule is in `Docs/rpc-design.md` §4: a verb that creates a node is verified on the node's CLASS, and reports the count the CONSUMER reads. Not moved to `DONE` — no separate tester has re-run this, and the full suite was not re-measured for this pair of commits.
