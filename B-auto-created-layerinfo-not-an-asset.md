---
id: B-auto-created-layerinfo-not-an-asset
title: "landscape.create_procedural_terrain's auto-created ULandscapeLayerInfoObject is outered to the ALandscape actor, so UObject::IsAsset() returns false and it can never appear in the Asset Registry, in asset.list, or in any picker — a shape the engine has nowhere, since UE::Landscape::CreateTargetLayerInfo always makes a real /Game package and the editor's own paint tool refuses to paint rather than auto-create; and the remedy the verb's own warning prescribes cannot be carried out, because no PinWright verb binds an existing LayerInfo to a target layer"
status: OPEN
severity: Medium
category: bug
tags: [landscape, create_procedural_terrain, layerinfo, landscapelayerinfoobject, asset-registry, packaging, outer, engine-factory-bypass, createtargetlayerinfo, material-authoring, add_landscape_layer, unperformable-workaround, no-binding-verb, discoverability]
encounters: 1
lastSeen: 2026-08-30T17:25:00+03:00
---

# The object exists, the asset does not, and the prescribed cure is not reachable

When `landscape.create_procedural_terrain` finds a material target layer with no
`ULandscapeLayerInfoObject`, it creates one — outered to the landscape **actor**:

    Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp:2779-2781

```cpp
ULandscapeLayerInfoObject* NewLayerInfo = NewObject<ULandscapeLayerInfoObject>(
    Landscape, FName(*FString::Printf(TEXT("LayerInfo_%s"), *LayerName)),
    RF_Public | RF_Transactional);
```

`Landscape` is the `ALandscape*` resolved at `:2552` by `FindLandscapeByNameOrPath` (declared
`:121-123`) and never reassigned between there and `:2779`. So the outer is an **actor instance**,
whose own outer chain is `ULevel` -> the `.umap` package (or, under One File Per Actor, the actor's
external package). The object is nested inside the level, not stored as anything.

## It can never be an asset — proven at the engine's own predicate, not inferred

`UObject::IsAsset()`, `C:/UE_5.8/Engine/Source/Runtime/CoreUObject/Private/UObject/Obj.cpp:2760`:

- `:2763` — `bHasValidObjectFlags` requires `!RF_Transient`, `!RF_ClassDefaultObject`, `RF_Public`.
  The auto-created object **passes** this.
- `:2777-2778` — *"Don't count objects embedded in other objects"*:
  `UPackage* ObjectPackage = Cast<UPackage>(GetOuter());` — null, the outer is an `AActor`.
- `:2780-2784` — falls back to `GetExternalPackage()`, also null (`RF_HasExternalPackage` is set
  only by an explicit `SetExternalPackage`, which this path never calls).
- `:2793` — `return false;`

`asset.list` is a pure Asset Registry query — registration
`Handlers/Asset/AssetManageHandler.cpp:1086`, `FARFilter` `:1142`, `ScanPathsSynchronous` `:1165`,
`GetAssets(Filter, AssetList)` `:1183` — so an object that is not an asset is not merely hard to
find there, it is **unreachable by construction**, along with `asset.search`, `asset.get`, any
content-browser picker, and any reference from another package a user could author by hand.

**A correction worth carrying, because it changes the fix:** the absent `RF_Standalone` is *not*
why this fails. `IsAsset()` never consults that flag. `RF_Standalone` means "keep alive when
unreferenced" (`ObjectMacros.h:591`), and adding it would change nothing here. The outer is the
whole story.

## The engine has this shape nowhere

**Its factory always makes a real package.** `UE::Landscape::CreateTargetLayerInfo` —
declared `C:/UE_5.8/Engine/Source/Runtime/Landscape/Public/LandscapeUtils.h:321` and `:330`
(both `LANDSCAPE_API`), defined `Private/LandscapeUtils.cpp:277` (2-arg) and `:286` (3-arg):

| | `CreateTargetLayerInfo` (`LandscapeUtils.cpp:286-321`) | `LandscapeHandler.cpp:2779-2781` |
|---|---|---|
| outer | `UPackage* Package = CreatePackage(*PackagePath)` `:295`, `check` `:297` | the `ALandscape` actor |
| a real `.uasset` | yes, unique name via `GetLayerInfoObjectPackageName` `:255-275` | no |
| project-default template | `Settings->GetDefaultLayerInfoObject().LoadSynchronous()` `:290`, `DuplicateObject` `:301` | never consulted |
| flags | `RF_Public\|RF_Standalone` at `:307`, then `SetFlags(RF_Transactional)` at `:308`, with the comment at `:306` explaining that passing `RF_Transactional` to `NewObject` makes the asset mark itself garbage on Undo | `RF_Public\|RF_Transactional` passed straight in |
| debug colour | `SetLayerUsageDebugColor(GenerateLayerUsageDebugColor(), ...)` `:313` | never set |
| registry | `FAssetRegistryModule::AssetCreated(LayerInfo)` `:316`, `Package->MarkPackageDirty()` `:317` | none |

`grep -rn "CreateTargetLayerInfo" Plugins/PinWright/Source/` returns nothing: the factory is not
called anywhere in the plugin. (`LandscapeUtils.h` is included at
`Handlers/Material/MaterialAuthoringHandler.cpp:64`, but only for
`RetrieveTargetLayerNamesFromMaterials`, used at `:3527`.)

**And the editor's own paint tool does not auto-create at all — it refuses to paint.**
`C:/UE_5.8/Engine/Source/Editor/LandscapeEditor/Private/LandscapeEdMode.cpp:4335-4342`,
`FEdModeLandscape::CanEditTargetLayer`:

```cpp
if (CurrentToolTarget.TargetType == ELandscapeToolTargetType::Weightmap && InLayerInfo == nullptr)
{
    if (Reason)
    {
        *Reason = NSLOCTEXT("UnrealEd", "LandscapeNeedToCreateLayerInfo",
            "This layer has no layer info assigned yet. You must create or assign a layer info before you can paint this layer.");
    }
    return false;
}
```

The user is instead routed through the Target Layers panel: a modal `SDlgPickAssetPath` titled
*"Create New Landscape Layer Info Object"*
(`LandscapeEditorDetailCustomization_TargetLayers.cpp:2333-2344`) -> `CreateTargetLayerInfoAsset`
`:2348` -> `UE::Landscape::CreateTargetLayerInfo` `:2354` -> `CreateTargetLayerSettingsFor` `:2363`
-> `GEditor->SyncBrowserToObjects` `:2368` -> `UpdateTargetLayer` `:2371`. The same shape appears
at `LandscapeEditorObject.cpp:950-971` and `..._ImportLayers.cpp:414`.

So this is not a private variant of an engine flow. **There is no engine path that produces a
LayerInfo outside a package**, and the one engine path that faces this exact condition chooses to
stop and ask rather than to invent one.

## What actually breaks — and one thing that does not

**The packaging is NOT silent, and a filing that said so would be wrong.** The verb warns on every
auto-create, `LandscapeHandler.cpp:2844-2847`:

> Target layer '%s' had no ULandscapeLayerInfoObject; one was created inside the landscape actor's
> package rather than as a shared /Game LayerInfo asset. Create a proper asset with
> `material.authoring.add_landscape_layer` and assign it if this layer is shared between landscapes.

and publishes `layerInfoAutoCreated` at `:3101`. A tester has seen it fire
(`B-create-procedural-terrain-paints-nothing` `#3`).

**The defect is that the remedy that warning prescribes cannot be carried out through this
plugin.** Two independent blockers, both re-derived:

1. **Nothing binds an existing LayerInfo to a target layer.** `grep -rn
   "CreateTargetLayerSettingsFor\|UpdateTargetLayer\|AddTargetLayer\|LayerInfoObj *="` across
   `Source/PinWright/Private/Handlers/` (tests excluded) hits only inside the auto-create branch
   itself (`LandscapeHandler.cpp:2810-2837`) and comments. `material.authoring.add_landscape_layer`
   (`MaterialAuthoringHandler.cpp:3150-3230`) creates the asset — correctly, in a package, at
   `:3167`/`:3174-3175` — and binds it to **no landscape**: that body contains no `ALandscape`, no
   `ULandscapeInfo`, no `CreateTargetLayerSettingsFor`, no `UpdateTargetLayer`. So the word
   *"assign"* in the warning names an operation that does not exist on the RPC surface.
2. **That asset does not reach disk either.** `add_landscape_layer`'s `save: true` is a bare
   `MarkPackageDirty()` at `MaterialAuthoringHandler.cpp:3217-3221` with no
   `SavePackage`/`save_asset` — which is `B-material-authoring-save-no-disk-write` (IN-REVIEW,
   Critical), whose own citations `:2576-2582` have drifted; the live line is `:3220`. That is
   that ticket's defect, not this one's, and is named here only because it is the second reason the
   prescribed remedy fails today.

**Reuse across landscapes** is therefore blocked — but the usual reason given for it is wrong and
should not be repeated: `RF_Public` means precisely *"Object is visible outside its package"*
(`ObjectMacros.h:590`), so nothing in the flags forbids a cross-package hard reference. It is
unreusable because it is not in the Asset Registry (so no verb or picker can name it) and because
of blocker 1 above.

**The claim I was handed about weight loss is scoped down deliberately.** Its source
(`Docs/map/vegetation-test-level.md:295-297`, host `EAContentExamples58`) says *"A paint that is
**never saved** therefore loses both the LayerInfo and the weight on the next level load"* — which
is true of anything unsaved and is a property of not saving, not of the packaging. The mechanism by
which it would matter is real and is worth recording for whoever fixes this:
`FWeightmapLayerAllocationInfo::LayerInfo` is a hard serialized `UPROPERTY`
(`Runtime/Landscape/Classes/LandscapeComponent.h:145-146`), and `ULandscapeComponent::FixupWeightmaps`
(`Runtime/Landscape/Private/LandscapeEdit.cpp:1172`) adds any allocation with `!Allocation.LayerInfo`
to `LayersToDelete` at `:1218-1221`, ending in `DeleteLayerAllocation` (`Landscape.cpp:3199`,
`RemoveAt` at `:3220` and `:3227-3230`) — so a lost LayerInfo deletes the weight outright rather
than orphaning it. **But whether a *saved* paint keeps them is NOT established here, in either
direction**, and this ticket does not assert it. A byte-level name scan of
`Content/Maps/PW_VegetationTest.umap` finds `LayerInfo_Grass` and `LayerInfo_Rock` present and
`WeightmapLayerAllocations` / `WeightmapTextures` absent, which reproduces
`B-paint-layer-destroys-other-layer-weights` `#6` and carries `#6`'s exact limitation: a name-table
hit is not proof of an export, and settling it needs an export-table parse nobody has done.

## Nothing else surfaces the choice either

- The registered summary (`LandscapeHandler.cpp:2463-2544`) mentions LayerInfo once, at `:2472`,
  and only about *orphaned* allocations.
- `Plugins/PinWright/Docs/wiki-src/landscape.md:237` lists `layerInfoAutoCreated` as a bare field
  name; the response-field table at `:243-248` omits it;
  `Saved/PinWright/wiki/landscape.create_procedural_terrain.md:41` inherits the omission.
- `Source/PinWright/Private/Tests/Environment/TestLandscapePaintLayerHonesty.cpp` contains no
  `GetOuter`, `GetPackage`, `IsAsset` or `/Game`, so no test asserts on the packaging in either
  direction.
- The warning's own wording is slightly off in a way worth fixing alongside: it says *"inside the
  landscape actor's package"*, but the object is inside the **actor**, whose package is the level.

## Ask

Pick one of two shapes; both are defensible and they differ in how much they change the verb's
contract.

1. **Route through the engine factory** — `UE::Landscape::CreateTargetLayerInfo(LayerName, Path,
   FileName)` (`LandscapeUtils.h:330`), then `LandscapeInfo->CreateTargetLayerSettingsFor(LayerInfo)`
   as the code already does at `:2815`. This produces a real `/Game` asset, registers it
   (`AssetRegistryModule::AssetCreated`), honours the project's `DefaultLayerInfoObject` template,
   sets the debug colour, and gets the `RF_Transactional`-after-construction ordering right. Needs
   a new optional `layerInfoPath` parameter (default something like `/Game/Landscape/Layers`) and
   an actual package save, since a created-but-unsaved asset is the same failure in a new place.
2. **Refuse, as the engine's own paint tool does** (`LandscapeEdMode.cpp:4335-4342`), naming
   `material.authoring.add_landscape_layer` — *only once a verb exists that can bind the result*,
   otherwise the refusal is a dead end.

**Either way, a binding verb is the prerequisite**, and it is the smaller piece of work:
`ULandscapeInfo::CreateTargetLayerSettingsFor` is already called at `LandscapeHandler.cpp:2815`, so
the call is known-good in this file. Without it, the sentence the verb prints at `:2844-2847`
stays unperformable.

Docs in the same change: the registered summary, `Docs/wiki-src/landscape.md:237` and its
response-field table `:243-248` should say what `layerInfoAutoCreated: true` means for
discoverability, and the warning text should say "inside the landscape actor" rather than "the
landscape actor's package".

## Distinct from

- **`B-paint-auto-created-layer-never-weight-blended`** (OPEN, High) — same `NewObject`, different
  consequence: that ticket is about the object's `BlendMethod` being `None`. **It explicitly
  carves the packaging out of its own scope**, in its § *Fix*: *"`LandscapeHandler.cpp` creates the
  object with outer `Landscape` and no `/Game` package **on purpose** ... the blend-method half is
  separable from that choice — do not let the packaging difference block the fix."* That is a
  disclaimer, not coverage, and it is why this is a new ticket rather than an encounter there:
  bumping its `encounters` would tell the picker `BlendMethod == None` was observed again (it was
  not) and would hand its fixer two contradictory instructions. Neither fix closes the other — that
  ticket accepts a bare `SetBlendMethod(FinalWeightBlending)` on the raw object, which leaves the
  outer untouched, and conversely creating a real asset on this host still yields `BlendMethod
  None` because `DefaultLayerInfoObject` is unset. **Sequencing note for whoever takes both:** that
  ticket's seat-1 fix (route through `CreateTargetLayerInfo`) closes this one too, so doing seat 1
  once is strictly cheaper than doing seat 2 and then this.
- **`B-add-landscape-layer-noweightblend-inverted-default`** (OPEN, Medium) — the *other* creation
  site (`MaterialAuthoringHandler.cpp:3174-3175`), which does make a real package. About its
  `SetBlendMethod` sitting inside a `TryGetBoolField` guard, not about packaging.
- **`B-material-authoring-save-no-disk-write`** (IN-REVIEW, Critical) — owns the fact that
  `add_landscape_layer`'s asset never reaches disk. Cited above as the second reason the prescribed
  remedy fails; not re-filed.
- **`B-layer-paint-doc-claims-blend-group-write`** (OPEN, Medium) — pure docs about blend-group
  attribution; asks for no code change to the paint path.
- **`B-paint-blanks-unallocated-layers-component-wide`** (OPEN, High),
  **`B-paint-census-counts-texels-not-weight`** (OPEN, Medium),
  **`B-paint-erases-orphaned-layer`** (IN-REVIEW, High),
  **`B-paint-layer-destroys-other-layer-weights`** (IN-REVIEW, Critical) — allocation, census and
  weight-destruction behaviour on the same verb. None touches where the object lives.
- **`B-create-procedural-terrain-paints-nothing`** (DONE, High) — the original false-success
  ticket; its `#3` is the only place on the board where a tester recorded seeing this warning fire,
  as a passing detail.

Board-wide `grep -ril "LayerInfo\|layer_info\|CreateTargetLayerInfo"` over the root tickets returns
ten files, all listed above. **No ticket owns the packaging choice.**

## Severity

**Medium**, on the rubric's soft-blocker band — *"doable, but only via a documented workaround, a
source dive, or many extra calls"*. A caller who needs a shared, discoverable target layer has to
leave the plugin for Python or the editor UI.

**The High reading is real and is declined with its argument.** The rubric's *"hard blocker with no
workaround ... so a reasonable task is impossible"* fits uncomfortably well: the workaround the verb
itself prints is not performable, because no verb binds an existing LayerInfo and because
`add_landscape_layer`'s asset does not reach disk. I decline High for two reasons. First, half of
what makes the remedy unperformable belongs to `B-material-authoring-save-no-disk-write`
(IN-REVIEW, Critical) — rating this High would double-count a defect already ranked above it.
Second, the High band's other clause is *silent* wrong data, and nothing here is silent: the verb
warns at `:2844-2847` and publishes `layerInfoAutoCreated`. Being told the truth and being unable to
act on it is worse ergonomics than a soft blocker, but it is not the class the High band names.

**Reach modifier declined.** `landscape.create_procedural_terrain` is a rare edge path, which by the
rubric argues a bump down to Low. Declined: Low is *"pure friction — docs, discoverability, naming,
cosmetic"*, and this is not friction — the auto-created object is unreachable by every asset verb
the plugin has, and the documented escape from it does not work. It stays Medium.

**Not Critical.** Nothing crashes. No asset is corrupted: the level's own weight data is written
normally, and a saved level keeps the object as a level-embedded export. Whether that survives
`FixupWeightmaps` is untested and is deliberately not asserted here.

severity rationale: impact=soft blocker, with the hard-blocker reading declined for double-counting
and for not being silent (Medium) x reach=rare path, bump down to Low declined because this is not
friction -> Medium

## Not done

Plugin source was not modified. No live editor call was made against this path — the level is
shared and `landscape.create_procedural_terrain` is already recorded as destroying other layers'
weights (`B-paint-layer-destroys-other-layer-weights`, Critical), so re-running it to observe the
warning would have cost more than the observation is worth; the warning text and the
`layerInfoAutoCreated` field are read from source and corroborated by a tester's note on
`B-create-procedural-terrain-paints-nothing` `#3`. The export-table question in § *What actually
breaks* was not settled and is explicitly left open.

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
