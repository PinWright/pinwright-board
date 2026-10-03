---
id: B-auto-created-layerinfo-not-an-asset
title: "landscape.create_procedural_terrain's auto-create warning tells the caller to \"assign\" a shared LayerInfo made with material.authoring.add_landscape_layer, but no PinWright verb binds an existing ULandscapeLayerInfoObject to a landscape target layer, so the prescribed remedy cannot be carried out (and the warning misplaces the object in \"the landscape actor's package\")"
status: OPEN
severity: Medium
category: bug
tags: [landscape, create_procedural_terrain, layerinfo, landscapelayerinfoobject, asset-registry, packaging, outer, engine-factory-bypass, createtargetlayerinfo, material-authoring, add_landscape_layer, unperformable-workaround, no-binding-verb, discoverability]
encounters: 1
lastSeen: 2026-08-30T17:25:00+03:00
rice: [1, 2, 1, 2]
priority: 8
---

# The auto-create warning prescribes an "assign" step no verb performs

When `landscape.create_procedural_terrain` paints a material target layer that has no
`ULandscapeLayerInfoObject`, it creates a private one outered to the `ALandscape` actor
(`CreateLandscapePaintTargetLayerInfo`,
`Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp:2709-2741`). That is a documented
design choice — the comment at `:2689-2700` explains why the engine factory
`UE::Landscape::CreateTargetLayerInfo` (which always mints a `/Game` asset) is not used — and it means
the object is not an asset (`UObject::IsAsset()` requires a package outer), so `asset.list`, pickers
and other landscapes cannot reach it.

The verb then warns (`LandscapeHandler.cpp:3226-3227`):

> Target layer '%s' had no ULandscapeLayerInfoObject; one was created inside the landscape actor's
> package rather than as a shared /Game LayerInfo asset. Create a proper asset with
> material.authoring.add_landscape_layer and assign it if this layer is shared between landscapes.

The "assign" half is not on the RPC surface:

- `material.authoring.add_landscape_layer`
  (`Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:3268`) creates a packaged
  LayerInfo and binds it to no landscape.
- The only `CreateTargetLayerSettingsFor` call in `Handlers/` is inside the auto-create branch
  (`LandscapeHandler.cpp:3193`); no `landscape.*` verb (`create`, `sculpt`, `set_material`,
  `create_grass_type`, `edit`, `get_heights`, `create_procedural_terrain`, `audit_shape`,
  `flush_grass`) takes an existing LayerInfo for a target layer.

So a caller who needs one LayerInfo shared between landscapes is told to do something they cannot do
through PinWright. The warning is also imprecise: the object is inside the landscape **actor**, not
the actor's package.

**Workaround:** assign the asset in Landscape Ed Mode's Target Layers panel, or via `python.execute`.
(`add_landscape_layer`'s `save: true` only marks dirty, `MaterialAuthoringHandler.cpp:3350-3354`;
persist with `asset.save` — that defect is `B-material-authoring-save-no-disk-write`.)

**Fix:** either add a binding verb (e.g. `landscape.set_layer_info {landscape, layerName,
layerInfoPath}` that registers the LayerInfo for the target layer via
`ULandscapeInfo::CreateTargetLayerSettingsFor`, as `:3193` already does, and reads the binding back),
or, if a binding verb is out of scope, change the warning to name a route that exists (the Target
Layers panel) instead of an unperformable "assign". In both cases change "inside the landscape
actor's package" to "inside the landscape actor".

**Acceptance:** after an auto-create, the warning text names an action the caller can perform; if
the verb is added, binding a `/Game` LayerInfo made by `add_landscape_layer` to the layer makes a
subsequent `create_procedural_terrain` on that layer report `layerInfoAutoCreated: false` and paint
through the shared asset.

## History
- `#1-layerinfo-outered-to-the-actor-is-not-an-asset` `OPEN` reporter — Filed from the final triage
  sweep of a vegetation session on host `EAContentExamples58` (UE 5.8, editor build 13:32, plugin
  commit `d8f1bc32`). Every citation re-derived at HEAD rather than relayed. **The creation site:**
  `Handlers/Environment/LandscapeHandler.cpp:2779-2781` is `NewObject<ULandscapeLayerInfoObject>(
  Landscape, FName(...LayerInfo_%s...), RF_Public | RF_Transactional)` inside the `if (!LayerInfo)`
  branch at `:2777-2848`; `Landscape` is the `ALandscape*` from `:2552`
  (`FindLandscapeByNameOrPath`, declared `:121-123`) with no reassignment in between, so the outer
  is an actor instance and not a package. Exactly two `NewObject<ULandscapeLayerInfoObject>` exist
  in the plugin — this one and `MaterialAuthoringHandler.cpp:3174-3175`, which does use a real
  `CreatePackage` (`:3167`). **Why it can never be an asset:** `UObject::IsAsset()`
  (`C:/UE_5.8/.../CoreUObject/Private/UObject/Obj.cpp:2760`) passes the flag test at `:2763`, then
  fails `Cast<UPackage>(GetOuter())` at `:2778` and `GetExternalPackage()` at `:2783`, reaching
  `return false;` at `:2793`; `asset.list` is a pure registry query
  (`AssetManageHandler.cpp:1086`, filter `:1142`, `GetAssets` `:1183`). **The engine has no such
  shape:** `UE::Landscape::CreateTargetLayerInfo` (`LandscapeUtils.h:321`, `:330`;
  `LandscapeUtils.cpp:277`, `:286-321`) always does `CreatePackage` `:295`, honours
  `GetDefaultLayerInfoObject` `:290`/`:301`, sets `RF_Public|RF_Standalone` at `:307` with
  `RF_Transactional` added afterwards for the reason spelled out at `:306`, sets the debug colour
  `:313`, and calls `AssetRegistryModule::AssetCreated` `:316` — and the plugin never calls it; the
  editor's own paint tool refuses to paint a weightmap layer with a null LayerInfo
  (`LandscapeEdMode.cpp:4335-4342`) and routes the user to a create-asset modal
  (`LandscapeEditorDetailCustomization_TargetLayers.cpp:2333-2371`). **Four claims in the lead I was
  handed did not survive and are corrected in the body rather than repeated:** the outer is the
  actor, never "or its package"; the packaging is **not** silent — `LandscapeHandler.cpp:2844-2847`
  warns and `:3101` publishes `layerInfoAutoCreated`, so the sharper defect is that the warning's
  prescribed remedy is unperformable (no verb binds an existing LayerInfo — grep over `Handlers/`
  for `CreateTargetLayerSettingsFor|UpdateTargetLayer|AddTargetLayer|LayerInfoObj *=` hits only the
  auto-create branch at `:2810-2837`; and `add_landscape_layer`'s asset does not reach disk,
  `MaterialAuthoringHandler.cpp:3217-3221`, which is
  `B-material-authoring-save-no-disk-write`'s); the reuse claim is right in effect but wrong in
  reason, since `RF_Public` explicitly means "visible outside its package"
  (`ObjectMacros.h:590`); and the missing `RF_Standalone` is **not** why `IsAsset()` fails, since
  that flag is never consulted there. The "an unsaved paint loses the weight" claim is scoped down
  in the body to what its source actually says and the saved case is left explicitly unsettled.
  **Filed as a new ticket rather than an encounter on
  `B-paint-auto-created-layer-never-weight-blended`** because that ticket carves the packaging out
  of its own scope in as many words and instructs fixers not to let it block them; neither fix
  closes the other, and a sequencing note is recorded instead. Severity Medium, with the
  hard-blocker High reading stated and declined (it would double-count
  `B-material-authoring-save-no-disk-write`, and nothing here is silent), and the rare-path bump
  down to Low declined because this is not friction.
- `#2-rephrased` `OPEN` developer — Re-checked at PinWright `7230b41d`. The actor-outered LayerInfo is now a documented deliberate design (`LandscapeHandler.cpp:2689-2700`, creation `:2709-2741`) that adopts the project `DefaultLayerInfoObject` template, and the blend-method half moved to `B-paint-auto-created-layer-never-weight-blended`; so the "route through `CreateTargetLayerInfo`" and "refuse like Ed Mode" asks, the engine-factory comparison table and the `IsAsset()` / weight-loss discussion are dropped. Retitled to the remaining defect: the warning (`:3226-3227`) prescribes an "assign" no verb performs (only `CreateTargetLayerSettingsFor` call is the auto-create branch `:3193`). Citations refreshed from `:2779`/`:2844` to the above. Severity unchanged (Medium).
