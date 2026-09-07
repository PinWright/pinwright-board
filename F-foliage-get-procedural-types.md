---
id: F-foliage-get-procedural-types
title: "No verb reads back which UFoliageTypes a UProceduralFoliageSpawner drives, though foliage.create_procedural already contains the exact FProperty walk that would do it — so a tuning pass identifies the types by the _FT_<index> asset name the create verb happens to emit, which turns a positional naming convention into load-bearing API"
status: OPEN
severity: Medium
category: feature
tags: [foliage, procedural-foliage, spawner, readback, missing-verb, foliage-type, reflection, naming-convention, vegetation, premise-corrected]
encounters: 1
costly: 1
lastSeen: 2026-08-29T18:00:00+05:00
---

# The write path already walks the array. Nothing reads it back

`foliage.create_procedural` builds a `UProceduralFoliageSpawner` and one
`UFoliageType_InstancedStaticMesh` per entry. Afterwards there is no verb that answers *"which types
does this spawner drive?"* — which is the first question any tuning pass asks of a spawner it did
not create in the same call.

**Confirmed absent, mechanically.** `ProceduralFoliageSpawner` appears in exactly three places under
`Plugins/PinWright/Source/`: an include at
`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp:26`, the
construction at `:1364`, and a test
(`Private/Tests/World/TestFoliageCreateProceduralTypeConfigHonesty.cpp:225-226`) covering the same
verb. All six `foliage.*` verbs — `paint` (`:151`), `remove` (`:592`), `get_instances` (`:705`),
`add_type` (`:847`), `add_instances` (`:979`), `create_procedural` (`:1273`) — read placed instances
or write new ones; none reads a spawner's configuration. The only thing the create response says
about types is a restatement of the request:

```cpp
Resp->SetNumberField(TEXT("foliage_types_requested"), FoliageTypesArr->Num());
```
`FoliageHandler.cpp:1589`

**The implementation is already written, twenty lines up.** The create path reaches the private array
by `FProperty` reflection, with a comment explaining why:

```cpp
// Add to Spawner using Reflection (since FoliageTypes is private)
FArrayProperty *FoliageTypesProp = FindFProperty<FArrayProperty>(
    Spawner->GetClass(), TEXT("FoliageTypes"));
```
`FoliageHandler.cpp:1454-1456`

then `FScriptArrayHelper` at `:1458-1462` and
`FindFProperty<FObjectProperty>(FFoliageTypeObject::StaticStruct(), TEXT("FoliageTypeObject"))` /
`FindFProperty<FBoolProperty>(..., TEXT("bIsAsset"))` at `:1464-1478`. A reader is the same walk with
the assignment reversed.

## Premise correction — "unreachable from Python" did not survive re-derivation

This was reported as *"a `UProceduralFoliageSpawner`'s foliage types are unreachable from Python"*
(`Docs/map/vegetation-style-split.md` § Findings 2), on two observations: the
`FoliageTypeObject` structs returned by `spawner.get_editor_property('foliage_types')` have a
`to_dict()` of `{}`, and `get_editor_property('foliage_type')` on one raises *"Failed to find
property 'foliage_type'"*. Both observations are real. The conclusion drawn from them is very
likely wrong, and saying so is more useful than the ticket the uncorrected version would have been.

**The name tried does not exist; the member is `FoliageTypeObject`.**
`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/FoliageTypeObject.h:45-46` declares
`TObjectPtr<UObject> FoliageTypeObject`, so the Python spelling is `foliage_type_object`, not
`foliage_type`. *"Failed to find property"* is the correct answer to the question that was asked.

**And the property is not access-gated the way the conclusion assumed.**
`PropertyAccessUtil::CanGetPropertyValue`
(`C:/UE_5.8/Engine/Source/Runtime/CoreUObject/Private/UObject/PropertyAccessUtil.cpp:425-433`)
denies a read only when the property has none of `CPF_Edit | CPF_BlueprintVisible |
CPF_BlueprintAssignable`:

```cpp
if (!InProp->HasAnyPropertyFlags(CPF_Edit | CPF_BlueprintVisible | CPF_BlueprintAssignable))
{
    return EPropertyAccessResultFlags::PermissionDenied | EPropertyAccessResultFlags::AccessProtected;
}
```

`FoliageTypeObject` carries `EditAnywhere` (`FoliageTypeObject.h:45-46`), and so does
`UProceduralFoliageSpawner::FoliageTypes` (`ProceduralFoliageSpawner.h:41-42`) — which is why
`get_editor_property('foliage_types')` returned structs at all rather than raising. C++ `private:`
does not gate `get_editor_property`; `CPF_Edit` un-gates it.

**What the empty `to_dict()` does establish**, and it is the part worth keeping: the struct is
`USTRUCT()` and **not** `BlueprintType` (`FoliageTypeObject.h:13-14`), every member is under
`private:` (`:43`) with no Blueprint specifier, `UProceduralFoliageSpawner::FoliageTypes` is
`EditAnywhere` with no Blueprint specifier while its own siblings `NumUniqueTiles` (`:30-31`) and
`MinimumQuadTreeSize` (`:34-35`) are `BlueprintReadOnly`, and the C++ accessor `GetFoliageTypes()`
(`:62`) is a plain inline getter carrying no `UFUNCTION`. So the *scriptable* surface really is
empty, every natural spelling fails, and a caller gets an empty dict from the one call that would
normally enumerate what is there. **The reachability is a hidden editor-only property behind a name
nobody guesses; the discoverability is genuinely nil.** That is a real finding and it is the reason
the wrong conclusion was reasonable.

**Not settled here, and it is one line to settle:** whether
`fto.get_editor_property('foliage_type_object')` actually returns the `UFoliageType`. The source
above says it should. Nobody ran it. Whoever picks this ticket up should run it first, because a
working spelling turns the severity argument below from "no workaround" into "an undiscoverable
one", which is where it is already rated.

## Consequence — a positional naming convention became load-bearing

Because the readback was believed impossible, the zone C/D re-speciation identified every type by
the asset name `foliage.create_procedural` emits. That name is built at `FoliageHandler.cpp:1438-1439`:

```cpp
FString FTName =
    FString::Printf(TEXT("%s_FT_%d"), *AssetName, TypeIndex++);
```

with `AssetName = Name + "_Spawner"` (`:1359`), under the hardcoded `/Game/ProceduralFoliage`
(`:1357`, path built `:1440-1441`) — so the real string is `<Name>_Spawner_FT_<index>`, e.g.
`PW_ZoneC_Woodland_Spawner_FT_3`. **The index is positional and encodes nothing about the mesh.**

This reopens something the previous triage pass deliberately declined, and the reversal is narrow, so
state it exactly. `Docs/map/vegetation-findings-dossier.md` § H dropped E-9 (*"procedural foliage
types are named `<Spawner>_FT_<index>`, so an asset keeps a stale name if its mesh is later
changed"*) on the correct ground that the name never encodes the mesh, so a mesh swap cannot stale
it. **That reasoning is untouched and E-9 as written stays dropped.** What changes is the adjacent
hazard the same section recorded and did not file *"for want of a repro"*: `_FT_<n>` is positional,
so re-running `create_procedural` with reordered `foliageTypes` rewrites `_FT_0` to a different
mesh. The repro it wanted now exists — a real tuning pass that resolved types **solely** by that
name and wrote `Mesh` onto the assets it found. Under a reorder, that pass edits the wrong asset,
silently and with every response reporting success.

The naming ticket is still not worth filing on its own: renaming `_FT_<n>` does not help anyone, and
a stable name would only move the problem. **The fix for the hazard is this ticket** — give callers
a readback keyed on the spawner instead of on a string, and the convention stops being API.

## Proposed verb shape

**`foliage.get_procedural_types`** — `{assetPath | actorName}` → `{spawner, types: [{index,
assetPath, mesh, isAsset}], count}`.

- `assetPath` resolves a `UProceduralFoliageSpawner` directly; `actorName` resolves an
  `AProceduralFoliageVolume` (or any actor carrying a `UProceduralFoliageComponent`) and follows it
  to its spawner, using the same actor resolution `spatial.ground_instances` and `pcg.generate`
  already share.
- `index` is the array position, stated as such so the caller can see it is positional rather than
  identity — and so the `_FT_<n>` correspondence is documented as a *consequence* of ordering
  instead of being reverse-engineered from asset names.
- `mesh` comes from the resolved `UFoliageType_InstancedStaticMesh`, which is what a caller actually
  wants to key on. `isAsset` mirrors the `bIsAsset` flag the create path already reads at `:1464-1478`.
- **Extraction, not new code.** Lift `FoliageHandler.cpp:1454-1478` into a small shared helper and
  have `create_procedural` keep calling it, so the reader and the writer cannot drift about how the
  private array is addressed. That is the same "extract the lambda so the two verbs cannot drift"
  argument `F-resimulate-existing-foliage-volume` makes about the placed-instance count.

Optional and strictly better if cheap: report the same `types[]` block in
`foliage.create_procedural`'s own response, which today says only `foliage_types_requested`
(`:1589`) — a number the caller supplied. A create that returned the assets it made would remove the
need to guess their names at all, which is the immediate half of this problem.

## Related

- `F-procedural-foliage-simulation-knobs-unreachable` (OPEN, High) — the write side of the same
  assets. Its `property.set`-onto-`_FT_<n>` workaround depends on finding the `_FT_<n>` assets,
  which is exactly what this ticket makes safe. A fixer taking both gets a tuning loop that can
  address types by identity.
- `F-resimulate-existing-foliage-volume` (OPEN, Medium) — the third side: re-running the simulation
  after those writes. Its `#2` records the one property (`Mesh`) whose write is not inert, which is
  precisely the property the re-speciation wrote by name.
- `B-create-procedural-ignores-scale-and-normal-fields`, `B-create-procedural-density-writes-paint-density`
  — both reference `_FT_<n>` assets by name for the same reason.

## Severity

**Medium.** Impact class is the rubric's *"hard blocker with no workaround (a stub, a missing verb…)"*
band, which spans High and Medium, and it lands on Medium because a workaround exists in both
directions: the `_FT_<n>` convention resolves the assets today (that is how the measured pass did its
work, successfully), and the source above says a Python read is one correctly-spelled call away. The
rubric's Medium is *"doable, but only via a documented workaround, a source dive, or many extra
calls"* — this is the source-dive case, and the source dive is what produced the correction section
above.

**Explicitly not High**: nothing here is silently wrong on a normal path today. The create verb does
not lie, the spawner is correct, and the pass that used the naming convention got the right answer —
the hazard needs a *reorder* to bite, and no run has hit that.

**Explicitly not Low**: the Low band is friction where the doc fails to help. A caller who reasons
about this correctly still cannot get the answer from any verb, and the fallback they are pushed
onto is a positional string built by a different verb, which is API by accident rather than
discoverability by inconvenience.

**Reach modifier declined in both directions.** Procedural-foliage authoring is not an every-session
path, so no bump up. Not a rare edge either: reading a spawner's types is the first step of every
tuning iteration after the first, and the whole point of a procedural volume is that it gets tuned.
Medium stands unmodified.

## History
- `#1-no-readback-and-the-name-is-the-api` `OPEN` reporter — Filed from the zone C/D re-speciation
  of `PW_VegetationTest` (`Docs/map/vegetation-style-split.md` § Zone C, § Findings 2). ABSENCE
  confirmed mechanically: `ProceduralFoliageSpawner` occurs in exactly three places under
  `Plugins/PinWright/Source/` (`FoliageHandler.cpp:26`, `:1364`, plus one test at
  `TestFoliageCreateProceduralTypeConfigHonesty.cpp:225-226`), all inside
  `foliage.create_procedural`; none of the six `foliage.*` verbs reads a spawner's configuration,
  and the create response says only `foliage_types_requested` (`:1589`), which restates the request.
  The reader is already written as a writer: `FoliageHandler.cpp:1454-1456` finds the private
  `FoliageTypes` array by `FProperty` reflection under a comment saying why, with `FScriptArrayHelper`
  at `:1458-1462` and the per-struct member lookups at `:1464-1478`. **PREMISE CORRECTED, and the
  correction is the load-bearing part of this ticket.** The reported claim was "the foliage types are
  unreachable from Python". Both observations behind it are real — `to_dict()` on a
  `FoliageTypeObject` is `{}`, and `get_editor_property('foliage_type')` raises "Failed to find
  property" — but the conclusion is very likely false: the member is named `FoliageTypeObject`
  (`FoliageTypeObject.h:45-46`), so the Python spelling is `foliage_type_object`, and
  `PropertyAccessUtil::CanGetPropertyValue` (`PropertyAccessUtil.cpp:425-433`) denies a read only
  when a property has none of `CPF_Edit | CPF_BlueprintVisible | CPF_BlueprintAssignable`, while
  both `FoliageTypes` (`ProceduralFoliageSpawner.h:41-42`) and `FoliageTypeObject` carry
  `EditAnywhere`. C++ `private:` does not gate `get_editor_property`. What the evidence *does*
  support is that the scriptable surface is empty and undiscoverable: the struct is `USTRUCT()` and
  not `BlueprintType` (`FoliageTypeObject.h:13-14`), all members are private with no Blueprint
  specifier (`:43`), `FoliageTypes` has no Blueprint specifier where its own siblings
  `NumUniqueTiles` (`:30-31`) / `MinimumQuadTreeSize` (`:34-35`) do, and `GetFoliageTypes()` (`:62`)
  carries no `UFUNCTION`. NOT SETTLED, one line to settle, flagged for whoever picks this up:
  whether `fto.get_editor_property('foliage_type_object')` actually returns the type. CONSEQUENCE,
  which is why this is filed as a verb request rather than a doc note: believing the readback
  impossible, the pass resolved every type by the asset name the create verb emits —
  `<Name>_Spawner_FT_<index>` (`FoliageHandler.cpp:1438-1439`, `AssetName` at `:1359`, hardcoded
  `/Game/ProceduralFoliage` at `:1357`) — which is positional and encodes nothing about the mesh.
  **Narrow reversal of a prior decision, stated exactly:** the dossier's § H dropped E-9 (stale name
  after a mesh swap) on reasoning that is still correct, and E-9 as written stays dropped; what
  revives is the *adjacent* hazard the same section recorded and left unfiled for want of a repro —
  re-running `create_procedural` with reordered `foliageTypes` rewrites `_FT_0` to a different mesh,
  and there is now a real pass that resolved types solely by that name and wrote `Mesh` onto what it
  found, so a reorder would have edited the wrong asset with every response reporting success. The
  remedy is this verb, not a rename: a stable name would only relocate the problem. Proposed
  `foliage.get_procedural_types {assetPath|actorName}` → `{spawner, types:[{index, assetPath, mesh,
  isAsset}], count}`, implemented by extracting `:1454-1478` into a helper both verbs call so reader
  and writer cannot drift; plus, if cheap, echoing the same `types[]` from `create_procedural`
  itself. Rated **Medium** on the missing-verb band's lower half, because two workarounds exist (the
  `_FT_<n>` convention, which the measured pass used successfully, and a Python read that is one
  correct spelling away); not High, since nothing is silently wrong today and the reorder hazard has
  not been hit; not Low, since the fallback is a positional string built by another verb, i.e. API
  by accident. Reach declined both ways: procedural foliage is not an every-session namespace, but
  reading a spawner's types is step one of every tuning iteration after the first.
