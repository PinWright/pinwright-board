---
id: B-paint-erases-orphaned-layer
title: "landscape.create_procedural_terrain silently erases an ALREADY-orphaned layer's weight landscape-wide, and the all-layers census cannot see it because it enumerates the registration side only"
status: IN-REVIEW
severity: High
category: bug
tags: [landscape, create_procedural_terrain, weightmap, layer-paint, orphaned-allocation, target-layers, data-loss, census-blind, silent-wrong-data, detector, world-partition]
encounters: 1
lastSeen: 2026-08-29T00:00:00Z
---

# An already-orphaned layer is erased with no warning, and the census is blind to it by construction

Residual of **`B-paint-layer-destroys-other-layer-weights`** (Critical, OPEN), named in its own
`#3-layerinfo-unbinding-is-the-mechanism` as *"Remaining hazard, deliberately NOT fixed here"*. That
ticket fixed the **producer** — the handler now registers its auto-created `ULandscapeLayerInfoObject`
through `LandscapeInfo->CreateTargetLayerSettingsFor(...)` so the binding reaches
`ALandscape::TargetLayers` and survives `UpdateLayerInfoMap()`. It did not fix the **consumer**: a
landscape that is *already* in the orphaned state — weight allocated on components, the layer absent
from the registration — still loses that weight on the next paint, silently, over the whole landscape.

**Orphaned state, defined precisely.** A `ULandscapeComponent` carries an
`FWeightmapLayerAllocationInfo` that the merge's requested-layer set does not contain. Two shapes, and
each is hidden by a *different* line of the census:

| | Shape A | Shape B |
|---|---|---|
| `ALandscape::TargetLayers` | name absent entirely | name present, `LayerInfoObj == nullptr` |
| `ULandscapeInfo::Layers` | entry absent | entry present, null `LayerInfoObj` |
| Component allocation | present, live `LayerInfo` | present, `LayerInfo` live **or** nulled |
| Hidden by | the census enumerating `Layers` at all | the explicit `!Setting.LayerInfoObj` skip |

## Verified against current source

The `#3` fix is in the **working tree, uncommitted** (`CreateTargetLayerSettingsFor` appears 4× in
`Handlers/Environment/LandscapeHandler.cpp` and 0× in that file at `HEAD`), still unbuilt and unrun.
Everything below is read from that working tree.

**The erase, re-derived on 5.8.** `ALandscape::GetTargetLayerNames` (`LandscapeEdit.cpp:8323-8343`)
transforms `LandscapeInfo->Layers` under the predicate `InSettings.LayerInfoObj != nullptr`.
`PerformLayersWeightmapsBatchedMerge` takes exactly that set as `RequestedWeightmapLayerNames`
(`LandscapeEditLayers.cpp:5883`). `ReallocateLayersWeightmaps` (`:5676`) then walks each component's
allocations and removes every one the merge did not request (`:5747-5757`):

```cpp
TArray<FWeightmapLayerAllocationInfo>& ComponentBaseLayerAlloc = Component->GetWeightmapLayerAllocations();
for (int32 i = ComponentBaseLayerAlloc.Num() - 1; i >= 0; --i) {
  const FWeightmapLayerAllocationInfo& Alloc = ComponentBaseLayerAlloc[i];
  if (!ComponentLayerAlloc.Contains(Alloc.LayerInfo)) { bRemoved = true; ComponentBaseLayerAlloc.RemoveAt(i); }
}
```

Both shapes fail the `LayerInfoObj != nullptr` filter, so both are erased from **every** component of
**every** loaded proxy — independent of `region` and of `strength`. The only trace is
`UE_LOGF(LogLandscape, VeryVerbose, "ReallocateLayersWeightmaps - ... Removed: %d ...")` at `:5846`.

**`Layers` cannot rescue either shape.** `ULandscapeInfo::UpdateLayerInfoMapInternal`
(`Landscape.cpp:4538`) does `Swap(Layers, PreviousLayers)` at `:4583` and repopulates `Layers` by
iterating `LandscapeActor->GetTargetLayers()` and nothing else (`:4586-4596`) — it carries over only
`ThumbnailMIC`, and back-fills only `ALandscapeProxy::VisibilityLayer`. It does **not** harvest
component allocations, and it does **not** synthesise placeholder entries from the material's declared
names. The outer `UpdateLayerInfoMap(nullptr, false)` (`:4612`) calls the internal unconditionally.

**The census is blind, and it is blind twice.** `LandscapeHandler.cpp:2583-2586`:

```cpp
for (const FLandscapeInfoLayerSettings& Setting : LandscapeInfo->Layers) {
  if (!Setting.LayerInfoObj || Setting.LayerName == CensusVisibilityLayer) { continue; }
```

The loop's *source* is the registration map, so shape A is unreachable — the layer is not in `Layers`
at all. The `!Setting.LayerInfoObj` guard then drops shape B. `otherLayerTexelsLost` (`:2730`) sums
only over `Census` entries, so it reports **0** over a landscape-wide erase, exactly as
`texelsWithWeight == paintedTexels` used to report clean over the parent defect. The census is a
strictly larger read than the old one-layer readback and still measures only the set that survives.

**Nothing else in the plugin can see it.** `GetWeightmapLayerAllocations`, `WeightmapLayerAllocations`
and `FWeightmapLayerAllocationInfo` have **zero** occurrences anywhere in `Plugins/PinWright/Source`,
tests included. So do `DeleteLayer` and `RemoveTargetLayer`. The test helper `FindLayerInfo`
(`Tests/Environment/TestLandscapePaintLayerHonesty.cpp:198-212`) has the same registration-side
blindness, and no test in that file constructs or asserts on an orphaned state —
`RegionPaintLeavesOtherLayersOutsideRegion` (`:665`) detects the condition only *after* the merge has
destroyed the data, by `FindLayerInfo` returning null.

**Undo does not cover it.** The `FScopedTransaction` opened at `:2626` closes before
`SettleLandscapeLayers(Landscape)` at `:2687`; the merge that runs `ReallocateLayersWeightmaps` is
inside that settle. With `verify:false` or `skipFlush:true` the merge is deferred further still. So
the existing warning's advice — *"Undo this call (editor.undo) before painting again"* (`:2738`) — does
not apply to this loss, and `Docs/wiki-src/landscape.md`'s *"The paint runs inside a single
transaction, so `editor.undo` reverses it — including the sibling layers' weight"* is wrong for it.

**Two in-band docs overstate the census.** The registered `verify` param promises to *"census EVERY
layer over the FULL landscape extent"*, and `Docs/wiki-src/landscape.md:223` says *"It censuses
**every** layer"*. Both are false for orphans, and they are what an agent reads before trusting
`otherLayerTexelsLost == 0`.

## How a layer becomes orphaned — this is NOT only a migration concern

**The engine itself never checks.** `FLandscapeEditDataInterface::SetAlphaData`
(`LandscapeEditInterface.cpp:2106`) creates the allocation unconditionally
(`new (ComponentWeightmapLayerAllocations) FWeightmapLayerAllocationInfo(LayerInfo)`, ~`:2226`) with
**no** test that `LayerInfo` is in `TargetLayers`; same in `ULandscapeComponent::FillLayer`
(`:1557-1561`). Any tool that paints without first calling `CreateTargetLayerSettingsFor` mints an
orphan — which is precisely what the parent Critical's pre-fix handler did, and what any other
BP/plugin/tool doing the same still does.

Live, non-PinWright producers, all verified in engine source:

- **Renaming a target layer in the Target Layers panel.**
  `LandscapeEditorDetailCustomization_TargetLayers.cpp:1598-1621`, inside a transaction literally named
  *"Rename Target Layer"*: `RemoveTargetLayer(old)` → `AddTargetLayer(new, FLandscapeTargetLayerSettings())`
  → `UpdateLayerInfoMap()`. `ALandscapeProxy::RemoveTargetLayer` (`Landscape.cpp:7725-7746`) removes the
  map entry and the `OnLayerInfoChanged` delegate and **iterates no components at all**; the replacement
  entry carries a **null** `LayerInfoObj`. One rename produces shape A (old name, live allocations, no
  registration) and shape B (new name, registered, no layer info) simultaneously.
- **Deleting a target layer while any proxy is unloaded.** `ULandscapeInfo::DeleteLayer`
  (`LandscapeEdit.cpp:4942-4965`) is the *correct* path — it removes from `Layers`, calls
  `RemoveTargetLayer`, and calls `FLandscapeEditDataInterface::DeleteLayer(LayerInfo)` — but that last
  step iterates `LandscapeInfo->XYtoComponentMap` (`LandscapeEditInterface.cpp:1506-1529`), populated
  only by `RegisterActorComponent`. Components on **unloaded World Partition proxies are absent**, so
  their allocations survive while the landscape-inherited `TargetLayers` loses the entry. It also
  deletes from only **one** edit layer (`GetEditLayer()`, ctor at `:36-49`), leaving other edit layers'
  `LayerAllocations` behind. The panel's delete button (`TargetLayers.cpp:2377-2395`) inherits both gaps.
- **Clearing a layer's asset slot in the panel.** `OnTargetLayerSetObject(nullptr)`
  (`TargetLayers.cpp:2226-2247`) → `ULandscapeInfo::ReplaceLayer(LayerInfoObj, nullptr)`, whose
  `ReplaceLayerInternal` null case assigns `ComponentWeightmapLayerAllocations[FromLayerIdx].LayerInfo = nullptr`
  (`LandscapeEditInterface.cpp:1723`) — an allocation still holding a texture channel with a null
  `LayerInfo`, plus a null `TargetLayers` entry.
- **Undo/redo.** `ALandscapeProxy::TargetLayers` and each component's `WeightmapLayerAllocations` are
  transactional but live on **different objects**, often different packages; an undo that restores one
  and not the other orphans transiently. The engine concedes this: `ULandscapeComponent::PostEditUndo`
  bumps `CurrentVersion` and forces `FixupWeightmaps()` (`LandscapeEdit.cpp:942, 1020`) with
  `bUndoChangedWeightmapAllocs` tracking consumed at `LandscapeEditLayers.cpp:5725-5732`.
- **Pre-fix damage from the parent Critical** — but see the load-time caveat below, which limits how
  much of it actually survives to meet a paint.

**NOT producers, contrary to the obvious guesses:**

- De-declaring a layer in the **material** (`material.authoring.configure_layer_blend`,
  `landscape.set_material`) does not orphan it. The merge's requested set comes from `TargetLayers` via
  `Layers`; the material has no part in it. `ALandscapeProxy::PostEditChangeProperty`
  (`LandscapeEdit.cpp:6143-6189`) calls `UpdateLayerInfoMap()` and refreshes MICs but mutates no target
  layer. The layer becomes unpaintable (`LAYER_NOT_FOUND`) and unsampled by the shader; registration
  and allocation both survive and no weight is lost.
- Gizmo **copy/paste** (`LandscapeEdModeComponentTools.cpp:1364-1381`) and **grid-size change / WP split**
  (`LandscapeConfigHelper.cpp:309-335`) both *skip* layers absent from `LandscapeInfo->Layers`. They
  silently drop orphaned weight rather than propagating it — data loss, but not an orphan source.
- `ALandscapeProxy::Import` registers explicitly (`LandscapeEdit.cpp:3791-3803`). Safe.

**The engine's own migration will not rescue pre-fix damage.** `ALandscapeProxy::PostLoad` harvests
allocations via `RetrieveTargetLayerInfosFromAllocations()` into `TargetLayersForFixup`
(`Landscape.cpp:4814-4843`) only under `LinkerVersion < FFortniteMainBranchObjectVersion::FixupLandscapeTargetLayersInLandscapeActor`
(version 157). A package already saved by 5.5+ is past that gate and never re-harvests, and the harvest
reads only the **base** allocations (`GetWeightmapLayerAllocations()` default, `LandscapeEdit.cpp:3100-3118`),
never edit-layer data. 5.3/5.4 are unaffected throughout, because there `UpdateLayerInfoMapInternal`
still rebuilt `Layers` from allocations — this is a **5.5-5.8-only** defect.

**The load-time caveat that narrows this ticket's window, stated because omitting it would overstate.**
`ULandscapeComponent::FixupWeightmaps(const FGuid&)` (`LandscapeEdit.cpp:1172-1330`) is the engine's own
detector for exactly this state, and it runs once per proxy per editor session on first registration —
`if (Proxy->WeightmapFixupVersion != Proxy->CurrentVersion) Proxy->FixupWeightmaps();`
(`Landscape.cpp:6469-6472`), and both counters are plain non-`UPROPERTY` members
(`LandscapeProxy.h:896-898`) that reset every load. So an orphan **saved to disk is destroyed at load
time**, before any merge sees it. The consequence: this ticket's live window is the orphan created
**mid-session** (rename, unloaded-proxy delete, asset-slot clear, undo, a third-party paint) and painted
before the next load, plus anything on an **unloaded proxy** the fixup could not reach. Much pre-fix
damage will already have been eaten by `FixupWeightmaps` rather than by this verb.

That detector's classification loop (`:1216-1243`) is also the right model for ours — null `LayerInfo`,
or `!Landscape->HasTargetLayer(Allocation.LayerInfo)` with a name-match salvage into `LayersToRemap`,
otherwise `LayersToDelete`. Note its posture: it reports to **MapCheck at Info level** (`:1259-1263`,
`FMapErrors::FixedUpDeletedLayerWeightmap`; the source comment says *"no need for a warning as the fix is
automatic and trivial"*) and then **deletes**, and there is no code anywhere outside the legacy
`TargetLayersForFixup` path that pushes an allocation's `LayerInfo` back into `TargetLayers`. The paint
path is strictly quieter than this: no MapCheck entry, no log above `VeryVerbose`.

## The detector: what it costs, and when it can run

`ULandscapeComponent::GetWeightmapLayerAllocations()` is `LANDSCAPE_API` in all four overloads
(`Classes/LandscapeComponent.h:819-822`), const and non-const, with a `bool` selector (base vs. the
*currently editing* layer) and an `FGuid` selector (invalid GUID = base; valid = that edit layer's
`LayerAllocations`, falling back to base). `FWeightmapLayerAllocationInfo` offers `GetLayerName()`
(`:176`) and `IsAllocated()` (`:186`) for skipping unallocated entries. Iterate with
`ULandscapeInfo::ForAllLandscapeComponents` (`LandscapeInfo.h:252`, impl `LandscapeEdit.cpp:4746-4759`),
**not** `XYtoComponentMap` — the latter holds only fully-registered components and is keyed by section
base. `ULandscapeInfo::GetUsedPaintLayers(FGuid(), Out)` (`LandscapeEdit.cpp:4999-5015`) is a ready-made
enumerator for the base pass. The engine's own ten-line walk,
`ALandscapeProxy::RetrieveTargetLayerInfosFromAllocations` (`LandscapeEdit.cpp:3100`), carries **no**
`LANDSCAPE_API` (its neighbours at `LandscapeProxy.h:1360-1368` do) and `ALandscapeProxy` is not
class-exported (`LandscapeProxy.h:450`), so it is not linkable from PinWright and must be reimplemented.

**Cost is negligible, and the comparison is the argument.** The walk is
`components × allocations-per-component` pointer comparisons against a `TSet` — a pure in-memory
traversal of resident `TArray`s of 10-byte structs. Nothing dereferences a `UTexture2D`, touches source
payload, streams, or hits disk. (`FixupWeightmaps` does pay I/O — `WeightmapTexture->ConditionalPostLoad()`
at `:1201-1204` — but only in its *repair* path; a read-only detector skips it.) Scale: a 505×505
landscape is 64 components, a 2017×2017 is 1024, large WP landscapes low thousands; 2-8 allocations each.
So 10³-10⁴ struct reads and as many set lookups — microseconds to low milliseconds. The census already in
the call does two `GetWeightDataFast` passes over the full extent *per layer*, up to 4,194,304 texels
each. The detector is roughly four orders of magnitude cheaper than the thing it completes.

**It can run before the write, and it needs nothing the census needs.** Its verdict is defined against
`TargetLayers`, which `UpdateLayerInfoMap()` (`:2415`) only *reads*, so the answer is identical before or
after that call and anywhere before the transaction at `:2626`. It requires no `SettleLandscapeLayers`
(it reads allocation arrays, not composited weightmaps) and it must **not** be gated on `verify` or on
`MaxCensusTexels` (`:2544`) — the two things that switch the census off are cost-driven and do not apply.

**Honest coverage limit:** `ForAllLandscapeComponents` delegates to `ForEachLandscapeProxy`
(`LandscapeInfo.h:441`), so the detector sees **loaded proxies only** — the same blind spot that makes
unloaded-proxy deletes an orphan source in the first place. The detector cannot promise "no orphans", only
"no orphans among loaded components", and its response field must say so rather than report a clean zero.

**Warning is not enough.** The parent ticket's proposal was "warn before the write". That repeats the
family's own failure mode one level up: the loss is landscape-wide, the transaction does not cover it (see
above), and there is no repair verb in the plugin to run afterwards, so a warning tells the caller about
data that is already gone and cannot be recovered by the remedy the adjacent warning recommends. The
plugin's convention for a destructive precondition is to **refuse with a dedicated code plus an explicit
opt-in**, and the existing precedents are all for *less* destructive outcomes: `landscape.sculpt` refuses a
no-op with `LANDSCAPE_SCULPT_NO_CHANGE` unless `allowNoChange` is passed (`LandscapeHandler.cpp:962`, gate
`:1390`), and `datatable` refuses to drop rows without `force`, via `ROWS_PRESENT_FORCE_REQUIRED`
(`DataTableAuthoringHandler.cpp:661, 684`).

## Fix

1. **Detect, before the write.** Between the layer resolution (`:2504`) and the census (`:2547`), walk
   `LandscapeInfo->ForAllLandscapeComponents` and, for each component's
   `GetWeightmapLayerAllocations()` (const, base overload), collect allocations whose `LayerInfo` fails
   `Landscape->HasTargetLayer(Allocation.LayerInfo)` (shape A) or is null (shape B), skipping
   `ALandscapeProxy::VisibilityLayer` and `!IsAllocated()` entries — mirroring the engine's own
   classification at `LandscapeEdit.cpp:1216-1243`. Optionally repeat per edit layer via
   `ForEachLayer` + the `FGuid` overload, as `FixupWeightmaps()` does at `:1160-1170`. Unconditional:
   not behind `verify`, not behind the texel cap.
2. **Refuse by default when a *live* orphan is found** (non-null `LayerInfo`), with a new error code
   (`LANDSCAPE_ORPHANED_LAYER_WEIGHT`) whose error data names each orphaned layer, its `LayerInfo`
   path and the component count carrying it, and whose message says the paint would erase that weight
   landscape-wide and that `editor.undo` will not restore it. Add one opt-in parameter — spelled to
   match the in-namespace precedent (`allowOrphanedLayerLoss`, default `false`) rather than a bare
   `force` — that downgrades the refusal to a `warnings[]` line and a response field.
3. **Report, never refuse, for a dead orphan** (null `LayerInfo`). Nothing can sample it,
   `GetWeightDataFast` has no object to read it by, and `FixupWeightmaps` deletes it as a matter of
   policy; refusing there would be pure obstruction. A `warnings[]` line is the correct weight.
4. **Report the orphans in the response** even on the opt-in path — an `orphanedLayers[]` array
   alongside `layersAffected[]`, plus a flag recording that only loaded proxies were scanned — so the
   number exists whether or not the call proceeded, and is never a clean zero it has not earned.
5. **Do NOT repair here.** Repair is *cheaper* than the parent ticket assumed — the allocation already
   holds a live `ULandscapeLayerInfoObject*`, so nothing needs recreating and
   `LandscapeInfo->CreateTargetLayerSettingsFor(Alloc.LayerInfo)` re-registers the exact object that is
   already bound — but it does not belong in a paint verb. It writes `ALandscape::TargetLayers`, a
   persistent actor property, as a side effect of painting an *unrelated* layer; it silently reverses a
   removal the user may have made deliberately (`RemoveTargetLayer` is exactly how the panel removes and
   renames); and it adopts a *more* aggressive policy than the engine, whose `FixupWeightmaps` salvages
   only by name-match and otherwise deletes. Put it behind a dedicated verb —
   `landscape.repair_target_layers`, mirroring that function's three-way delete/remap/re-register
   classification and reporting what it changed — and have the refusal in (2) name that verb as the
   remedy. A repair verb is also the only place that can sensibly require proxies to be loaded first.
6. **Correct the in-band overstatements**: the registered `verify` param and
   `Docs/wiki-src/landscape.md:223` both promise a census of *every* layer, and `:235` promises undo
   covers the sibling loss. Both need the orphan caveat, and the `otherLayerTexelsLost` warning at
   `:2738` should stop recommending `editor.undo` for a loss the transaction does not cover.
7. **Test it.** `Tests/Environment/TestLandscapePaintLayerHonesty.cpp` currently never builds a
   `ULandscapeLayerInfoObject` itself — every one comes from the handler's auto-create branch — so a new
   fixture is required: paint layer A, orphan it the way the editor's rename does
   (`Landscape->RemoveTargetLayer(A)` with no `DeleteLayer`), assert by direct
   `GetWeightmapLayerAllocations()` that the allocation survives, then paint layer B and assert the verb
   **refuses** and A's allocation is intact; and a second case asserting the opt-in proceeds, erases, and
   reports `orphanedLayers[]`. The test must not rely on `FixupWeightmaps` not having run.

## Severity

Impact = **Critical** by the README band — a write that loses asset data, here every texel of an
orphaned layer's weight across every loaded component, with the verb's own all-layers census reporting
`otherLayerTexelsLost: 0` over it and no transaction covering the loss. The census lie is independently a
**High** ("silent wrong data on a normal path — the caller trusts a result that is a lie and builds on
it"), and it lands on the one field the verb advertises as the reason to trust a multi-layer paint.

**Reach modifier: down one, to `High`.** Unlike the parent Critical — which fires unconditionally on the
second paint of a second layer, the first call that could possibly reveal it — this needs the landscape to
*already* carry an orphaned allocation, and the load-time `FixupWeightmaps` pass documented above will have
eaten much of the historical population before a paint ever reaches it. The live window is a mid-session
orphan (panel rename, unloaded-proxy delete, asset-slot clear, undo, third-party paint) or an unloaded
proxy. That is a real and reachable coincidence of two states rather than a guaranteed outcome, on a verb
that is itself infrequent. `High` rather than `Medium` because the census lie holds the floor on its own.
A fixer who finds orphaned landscapes common in this project's content should re-rate up to Critical; this
ticket surveyed none.

## Distinct from

- **`B-paint-layer-destroys-other-layer-weights`** (Critical, OPEN) — the parent. That ticket is the
  *producer* of the orphaned state and its `#3` fix stops the verb creating new orphans; this ticket is
  the *consumer*, and it is the hazard `#3` explicitly declined to fix. Independent: the parent's fix is
  necessary and does nothing for a landscape already in the state. Both must land before multi-layer
  terrain is safe on existing content.
- **`B-create-procedural-terrain-paints-nothing`** (DONE, High) — introduced the single-layer readback;
  its `Tests/Environment/TestLandscapePaintLayerHonesty.cpp` is where the new fixture belongs. Its `DONE`
  is untouched: it never claimed to look outside the requested layer.
- **`B-material-authoring-save-no-disk-write`** (IN-REVIEW, Critical) — its `#3-additional-cold-load`
  establishes that `material.authoring.add_landscape_layer` never writes its layer-info asset to disk. A
  landscape saved referencing one would load cold with a null `LayerInfoObj`, i.e. shape B. Cited as a
  route, not a duplicate.
- **`B-configure-layer-blend-wrong-nodes`** (IN-REVIEW, High) — authors the material's target-layer
  declaration. Explicitly **not** an orphan producer, per the refutation above.
- **`E-find-orphaned-nodes-misleading-verb`**, `B-decompile-orphan-pure-nodes-grafted`, and every other
  board ticket matching `orphan` — all Blueprint-graph or mesh-vertex orphans. Unrelated; named because
  the word collides on every dedup pass over this area.

## History
- `#1-orphan-erase-unhandled-and-census-blind` `OPEN` reporter — Source-read verification of the residual the parent Critical named in `#3-layerinfo-unbinding-is-the-mechanism` and deliberately left unfixed. Confirmed in the CURRENT working tree (the `#3` fix is uncommitted — `CreateTargetLayerSettingsFor` 4× in the worktree copy of `LandscapeHandler.cpp`, 0× at `HEAD` — and still unbuilt/unrun): the census loop at `:2583` enumerates `LandscapeInfo->Layers` and `continue`s on `!Setting.LayerInfoObj` at `:2586`, so it is blind to an orphaned allocation twice over — shape A never reaches the loop, shape B is skipped by the guard — and `otherLayerTexelsLost` at `:2730` therefore reports 0 over a landscape-wide erase. Re-derived the erase on 5.8 rather than trusting `#3`: `GetTargetLayerNames`' `LayerInfoObj != nullptr` predicate (`LandscapeEdit.cpp:8323-8343`), `RequestedWeightmapLayerNames` (`LandscapeEditLayers.cpp:5883`), the removal loop (`:5747-5757`), and its `VeryVerbose`-only trace (`:5846`). Read `UpdateLayerInfoMapInternal` (`Landscape.cpp:4538-4610`) in full: `Layers` is rebuilt PURELY from `LandscapeActor->GetTargetLayers()` (`:4583-4596`), with no allocation harvest and no material-name placeholders. **Central finding: this is NOT merely a migration concern for pre-fix content.** The Target Layers panel's RENAME action orphans deterministically — `RemoveTargetLayer(old)`, which iterates no components (`Landscape.cpp:7725-7746`), then `AddTargetLayer(new, FLandscapeTargetLayerSettings())` with a null layer info (`LandscapeEditorDetailCustomization_TargetLayers.cpp:1598-1621`). Three further live routes: deleting a target layer with WP proxies unloaded (`ULandscapeInfo::DeleteLayer` cleans only `XYtoComponentMap` and only the current edit layer, `LandscapeEdit.cpp:4942-4965` + `LandscapeEditInterface.cpp:1506-1529`); clearing a layer's asset slot (`ReplaceLayerInternal` assigns `Allocation.LayerInfo = nullptr`, `LandscapeEditInterface.cpp:1723`); and undo/redo, since `TargetLayers` and `WeightmapLayerAllocations` are transactional on different objects (engine concedes it at `LandscapeEdit.cpp:942, 1020`). Also established that `SetAlphaData` never checks registration (`LandscapeEditInterface.cpp:~2226`), so the parent bug was an instance of a general engine hazard. **Load-time caveat recorded deliberately because omitting it would overstate the ticket:** `FixupWeightmaps` (`LandscapeEdit.cpp:1172-1330`) is the engine's own detector for this state and runs once per proxy per session on first registration (`Landscape.cpp:6469-6472`; both version counters are non-UPROPERTY, `LandscapeProxy.h:896-898`), deleting orphans and reporting MapCheck **Info** — so an orphan saved to disk is destroyed at LOAD, and this ticket's live window is mid-session orphans plus unloaded proxies. That caveat is what moved the rating off Critical. **Two claims were checked and REFUTED rather than carried forward**: (a) that the paint verb's pre-existing-layer path could paint a `Layers` entry absent from `TargetLayers` — impossible on 5.5+, because `UpdateLayerInfoMap()` at `:2415` rebuilds `Layers` from `TargetLayers` immediately before the scan at `:2419`; (b) that de-declaring a layer in the material (`configure_layer_blend` / `set_material`) orphans it — the merge's requested set never consults the material, and `ALandscapeProxy::PostEditChangeProperty` (`LandscapeEdit.cpp:6143-6189`) mutates no target layer, so registration and allocation both survive. Gizmo paste (`LandscapeEdModeComponentTools.cpp:1364-1381`) and grid-size change (`LandscapeConfigHelper.cpp:309-335`) were checked too: they silently DROP unregistered layers, so they are data-loss paths but not orphan sources. Confirmed `ForEachLandscapeProxy` visits the parent `ALandscape` first, so `#3`'s `CreateTargetLayerSettingsFor` (`Landscape.cpp:4478`) writes the map the rebuild reads and is structurally sound. Detector cost assessed by inspection, not benchmark: `GetWeightmapLayerAllocations` is `LANDSCAPE_API` (`LandscapeComponent.h:819-822`), iterate via `ForAllLandscapeComponents` (`LandscapeInfo.h:252`) not `XYtoComponentMap`; a pure in-memory walk of ~10³-10⁴ structs, roughly four orders of magnitude cheaper than the `GetWeightDataFast` census already running, needing no settle and no texel cap, but covering LOADED PROXIES ONLY — which the response must state rather than report a clean zero. The engine's equivalent helper `RetrieveTargetLayerInfosFromAllocations` (`LandscapeEdit.cpp:3100`) is NOT exported and must be reimplemented. Dedup: grepped the board for `GetWeightmapLayerAllocations` (1 file — the parent), `TargetLayers` (2), `LayerInfoObj` (3), `weightmap` (10) and `orphan` (9, all Blueprint-graph or mesh-vertex); read the frontmatter of every landscape/layer-adjacent hit. Nothing covers the orphaned-allocation consumer. NOT DONE, and deliberately: nothing was compiled, no test was run, and no live editor was touched — a test suite is running in this checkout. The orphaned state was NOT reproduced at runtime; every claim here is source-read, and the rename-orphans-a-layer route in particular is derived from the panel's code path, not observed.
- `#2-allocation-side-census-and-refusal` `IN-REVIEW` developer — Fixed both halves in `Handlers/Environment/LandscapeHandler.cpp`. **Census now reads the ALLOCATION side.** New file-static `ScanLandscapeOrphanedWeightAllocations` walks `ULandscapeInfo::ForEachLandscapeProxy` -> `ALandscapeProxy::LandscapeComponents` -> `ULandscapeComponent::GetWeightmapLayerAllocations(false)` and flags every `IsAllocated()` entry whose `LayerInfo` is not in the set of non-null `LandscapeInfo->Layers[i].LayerInfoObj` pointers (shape A) or is null (shape B). The predicate is the merge's own and is compared by POINTER, not by name, because `ReallocateLayersWeightmaps` compares `ComponentLayerAlloc.Contains(Alloc.LayerInfo)` (re-read at `LandscapeEditLayers.cpp:5738-5757` on the 5.8 source on this host) and its requested set is `GetTargetLayerNames(true)` = `Algo::TransformIf(Layers, LayerInfoObj != nullptr)` (`LandscapeEdit.cpp:8330-8336`). `ALandscapeProxy::VisibilityLayer` is added to the survival set explicitly since the merge requests it with `bInIncludeVisibilityLayer = true`. Deliberately used `ForEachLandscapeProxy`, NOT `XYtoComponentMap`, and NOT `ALandscapeProxy::RetrieveTargetLayerInfosFromAllocations` (confirmed here that it carries no `LANDSCAPE_API` while its neighbours do, so it is not linkable and was reimplemented as the ticket said). **Refusal.** The scan runs unconditionally between `UpdateLayerInfoMap()` and the existing-layer resolution — before the auto-create branch and before the transaction, so a refusal mutates nothing — and is NOT gated on `verify` or on `MaxCensusTexels`. A live orphan refuses with the new `ErrorCodes::ERR_LANDSCAPE_ORPHANED_LAYER_WEIGHT` (added as a symbol to `Handlers/ErrorCodes.h`, not a raw literal, since this file is in the registry-adopting set), whose message names each orphaned layer + its LayerInfo path + component count, says the erasure is landscape-wide and that `editor.undo` does not cover it, and names the opt-in; error data carries `orphanedLayers[]`, `orphanedLayerCount`, `orphanScanComponents`, `orphanScanProxies`, `orphanScanLoadedProxiesOnly`, `worldPartitioned`. New declared param `allowOrphanedLayerLoss` (default false) downgrades it to a `warnings[]` line. A DEAD orphan (null `LayerInfo`) only warns, per the ticket — nothing can sample it and `GetWeightDataFast` has no object to read it by. **Never a clean zero:** `orphanScanComponents` / `orphanScanProxies` / `orphanScanLoadedProxiesOnly` travel on EVERY success too, and `orphanedLayers[]` travels on the opt-in path. **World Partition:** verified from the 5.8 source on this host that `ForAllLandscapeComponents` delegates to `ForEachLandscapeProxy` (`LandscapeEdit.cpp:4746`, `Landscape.cpp:6319`), which visits the parent `ALandscape` and every RESIDENT `ALandscapeStreamingProxy` — so per-proxy weight data on a partitioned landscape IS covered for loaded proxies; a partitioned world additionally emits a `warnings[]` line naming the proxy/component counts actually scanned and stating the verdict is about loaded components rather than about the landscape. **Not exercised at runtime on an actual WP landscape** — the regression tests build a monolithic 2x1-component landscape and the WP path is covered structurally (proxy iterator + flag + warning), not observed. **In-band overstatements corrected:** the `verify` param no longer promises a census of EVERY layer (it says registered layers and points at the orphan scan); the `otherLayerTexelsLost` warning no longer recommends `editor.undo`, because the loss happens in the merge inside the settle after the transaction closes; `Docs/wiki-src/landscape.md` gets the same three corrections plus an *Orphaned layer weight* block and a `LANDSCAPE_ORPHANED_LAYER_WEIGHT` error-table row. **NOT done, deliberately:** no repair, and the refusal does NOT name a `landscape.repair_target_layers` verb because that verb does not exist in this tree and naming it would be a fabricated remedy; it names the Target Layers panel and the opt-in instead. **Regression tests** in `Tests/Environment/TestLandscapePaintLayerHonesty.cpp`, both 5.5+ guarded (on 5.3/5.4 `UpdateLayerInfoMapInternal` still rebuilds `Layers` from allocations so the defect does not exist, and `RemoveTargetLayer` does not exist either): `PinWright.landscape.create_procedural_terrain.OrphanedAllocationRefusesPaint` paints layer A, orphans it with `Landscape->RemoveTargetLayer(A, /*bPostEditChange=*/false)` + `UpdateLayerInfoMap()` (the panel rename's mechanism), asserts through a new direct `GetWeightmapLayerAllocations` walk that the allocation SURVIVED the deregistration, then paints B and asserts the verb refuses `LANDSCAPE_ORPHANED_LAYER_WEIGHT` and A's allocations are still intact; `...OrphanedAllocationOptInReportsLoss` asserts the opt-in proceeds, reports `orphanedLayers[]` + the loaded-proxies-only flag + a warning, really does erase A's allocations, and that `otherLayerTexelsLost` still reads 0 over that erase — the census lie, pinned as an assertion. Both fail today: before the fix the refusal never fires and the merge destroys A. The fixture asserts BOTH halves of the orphaned state before painting, so a `FixupWeightmaps` pass eating the allocation fails the fixture rather than being blamed on the verb. Did NOT touch `RegionPaintLeavesOtherLayersOutsideRegion` or the parent's `CreateTargetLayerSettingsFor` registration; the scan finds nothing on that fixture because every layer the merge allocates there is registered. `check_test_ids.py` re-run: CLEAN, 4776 ids, no dot-prefix collisions. NOT COMPILED and NOT RUN — the orchestrator builds; brace/paren balance was verified against HEAD with a tokenizer instead.
