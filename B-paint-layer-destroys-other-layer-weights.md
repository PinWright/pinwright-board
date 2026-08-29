---
id: B-paint-layer-destroys-other-layer-weights
title: "landscape.create_procedural_terrain zeroes every OTHER weight layer across the WHOLE landscape, not just the painted region — and its own verification field reads only the layer it just painted, so texelsWithWeight == paintedTexels reports clean over the destruction"
status: OPEN
severity: Critical
category: bug
tags: [landscape, create_procedural_terrain, weightmap, layer-paint, data-loss, no-transaction, verification-blind, silent-wrong-data, vegetation, terrain, target-layers, layer-info-binding]
encounters: 2
lastSeen: 2026-08-29T18:00:00+05:00
---

# Painting one layer destroys the others, and the verb's own check cannot see it

`landscape.create_procedural_terrain` (`Handlers/Environment/LandscapeHandler.cpp:1981`) paints a
weight-blended layer over an optional heightmap-pixel region (`:1987`). Measured twice on a running
editor against a landscape whose material declares four target layers: after one call, **every layer
other than the one named held zero weight, across the entire landscape** — not merely inside the
`region` that was passed. Only one layer can carry weight at a time, which makes multi-layer terrain
painting unreachable through this verb. The vegetation test level had to ship Grass at strength 1.0
over the full extent with colour variation driven procedurally in the material, because the second
paint erased the first.

## The verification field is structurally incapable of catching it

The `verify` block reads back exactly one layer — the one just painted:

```cpp
LandscapeRead.GetWeightDataFast(LayerInfo, PaintMinX, PaintMinY, PaintMaxX, PaintMaxY,
                                ReadBack.GetData(), /*Stride=*/0);
```

`:2344-2345`, tallied at `:2346-2350` into `sampledTexels` / `texelsWithWeight` /
`texelsAtRequestedWeight` and reported at `:2380-2384`. `LayerInfo` is the requested layer and the
sample rectangle is the requested region, so the readback is bounded by both of the two axes along
which the damage escapes: it never looks at another `ULandscapeLayerInfoObject`, and it never looks
outside `PaintMinX..PaintMaxY`. `texelsWithWeight == paintedTexels` is therefore green in exactly the
run that wiped three other layers landscape-wide, and the warning at `:2356-2362` — which fires only
on `TexelsWithWeight == 0` for the painted layer — stays silent.

The readback is correct about what it measures. It measures the wrong set.

## The write is not in a transaction, so it is not undoable

`Handlers/Environment/LandscapeHandler.cpp` contains **one** `FScopedTransaction`, at `:659`, in
`landscape.create`. The paint path opens none: `PinWright::MarkLevelActorModified(Landscape)`
(`:2294`) is `Actor->Modify()` + `Actor->MarkPackageDirty()` (`EnvironmentDirtyUtils.h:61-66`), and a
bare `Modify()` outside a transaction records nothing for `editor.undo` to reverse. Combined with the
blind readback, a caller learns the other layers are gone only by opening Landscape Ed Mode, by which
point the only recovery is discarding the level's unsaved state.

## Mechanism: NOT established — two candidates, and the experiment that separates them

Stated as a hypothesis, not a finding. The call site is
`LandscapeEdit.SetAlphaData(LayerInfo, PaintMinX, PaintMinY, PaintMaxX, PaintMaxY, AlphaData.GetData(), RegionSizeX)`
(`:2316`) — the 8-argument overload
(`C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeEditInterface.cpp:2106`), taking the
default `PaintingRestriction = None`.

1. **Weight renormalization inside `SetAlphaData`.** The handler's own comment at `:2352-2354`
   already knows this happens — *"`SetAlphaData` renormalizes this layer against the others in the
   same blend"* — and cites it as the reason an intermediate strength reads back at a different
   value. At strength 1.0 that renormalization drives every sibling layer to zero. It explains
   destruction **inside** the region and does not by itself explain destruction outside it. Note that
   the engine's 10-argument overload (`:2378-2381`) accepts `bWeightAdjust` / `bTotalWeightAdjust`
   and then **discards both**, forwarding to the 8-argument form — so there is no engine-side knob to
   turn off, whichever overload a fix reaches for.
2. **The edit-layer recomposite.** The write is scoped to the default edit layer through
   `FScopedSetLandscapeEditingLayer EditingLayerScope(Landscape, EditLayerGuid, [Landscape] { Landscape->RequestLayersContentUpdateForceAll(); })`
   (`:2305-2318`), and `verify` then calls `SettleLandscapeLayers(Landscape)` (`:2333`) to drain it.
   `RequestLayersContentUpdateForceAll` recomposites the **whole** landscape's weightmaps from the
   edit-layer stack; if the sibling layers' weight lives in base data the recomposite treats as
   subordinate to the edit layer, they are zeroed everywhere the recomposite runs, which is
   everywhere. This is the candidate that explains the landscape-wide reach.

**The experiment that decides it.** Paint layer A at strength 1 over a small region, then repeat with
`verify: false` **and** `skipFlush: true` so neither `Flush()` (`:2318-2320`) nor
`SettleLandscapeLayers` (`:2333`) runs, sampling layer B inside and outside the region with
`GetWeightDataFast` before and after. If B survives until the settle, the cause is (2) and the fix
belongs in the edit-layer scope. If B is already zero outside the region before any settle, the cause
is (1) plus a component-range effect and the fix is a painting restriction.

## Fix

1. **Make the verb see its own destruction before making it non-destructive.** Extend the `verify`
   block into a whole-landscape, all-layers census: sample every `ULandscapeLayerInfoObject` in
   `LandscapeInfo->Layers` over the full extent before and after, and report `layersAffected[]`
   (name, texels-with-weight before, after) plus an aggregate `otherLayerTexelsLost`. That is a
   strictly larger read than the one at `:2344` and needs no new engine API. Non-zero
   `otherLayerTexelsLost` is a `warnings[]` line at minimum; once the mechanism is known, a refusal.
2. **Open an `FScopedTransaction` around the paint**, per the plugin convention that every mutation
   is wrapped. Undo does not undo a bare `Modify()`, and this is the verb where that costs the most.
3. Then fix the destruction itself, guided by the experiment above.

## Same shape as

`B-foliage-paint-does-no-ground-projection`, `B-ground-probe-hits-hull-not-render`,
`B-create-grass-type-addzeroed-never-renders` — the call succeeds, every number it reports is
correct, and the output is wrong because the deciding number was never reported. This instance is the
sharpest of the family: the missing number is not merely unreported, the verb *has* a verification
block, it was added deliberately, and its sampling bounds exclude the damage on both axes.

## Distinct from

- **`B-create-procedural-terrain-paints-nothing`** (DONE, High) — same verb, and the ticket that
  **introduced** the `texelsWithWeight` readback this one proves insufficient. A fixer must extend
  that block rather than write a new one, and its tests in
  `Tests/Environment/TestLandscapePaintLayerHonesty.cpp` are where the census assertions belong. Its
  DONE is not contradicted: nothing in it, its history, or its tests ever looks at a layer other than
  the requested one or at a texel outside the requested region, so it verified exactly what it
  claimed.
- **`B-configure-layer-blend-wrong-nodes`** (IN-REVIEW, High) — the upstream verb that authors the
  `LandscapeLayerBlend` this one paints into, and part of why multi-layer terrain was already hard to
  reach.
- **`E-create-procedural-terrain-no-label-set`** / **`E-create-procedural-terrain-no-material-echo`**
  — a DIFFERENT verb despite the name (`environment.build.create_procedural_terrain`, a procedural
  mesh actor, `EnvironmentHandler.cpp:747`). Named here because the slug collision costs a reader
  time on every dedup pass over this area.
- **`B-landscape-set-material-stale-mics`**, **`B-landscape-sculpt-stale-bounds`** — same handler,
  material and height axes, no weight involvement.

## Severity

Impact = **Critical**. The README's Critical band is "a write that corrupts or loses asset data", and
this is that literally: persisted weightmap data for every unrelated layer, destroyed outside the
region the caller named, with no transaction to reverse it and no reported field that reveals it. It
sits a band above `B-create-procedural-terrain-paints-nothing`'s High because that verb wasted a
call; this one takes away work the caller never put at risk.

**Reach modifier declined, and the decline is the part worth arguing.**
`landscape.create_procedural_terrain` is one of eight `landscape.*` verbs and the only path to layer
painting, so there is no case for a bump up. The bump *down* is the tempting one — most sessions
never paint terrain — and it is declined because rarity of use is not rarity of path: every caller
who paints a second layer hits this on the second call, which is the first call that could possibly
reveal it. Critical stands unmodified.

## History
- `#1-other-layers-zeroed-landscape-wide` `OPEN` reporter — Measured twice against a running editor during a vegetation test-level build, on a landscape whose material declares four target layers; this is not a source reading. After one `landscape.create_procedural_terrain` call at strength 1.0 over a sub-region, every layer other than the named one held zero weight over the FULL landscape extent, and the response was clean — `paintedTexels == texelsWithWeight`, no `warnings[]`. The second observation reproduced it. Every mechanism line cited above was re-derived against HEAD in this checkout: registration `LandscapeHandler.cpp:1981`, region param `:1987`, edit-layer scope `:2305-2318`, `SetAlphaData` `:2316`, single-layer single-region readback `:2344-2350`, zero-weight-only warning `:2356-2362`, response fields `:2379-2384`, and the absence of any `FScopedTransaction` in the file except `:659`. The destruction MECHANISM is deliberately left unestablished: two candidates are named with the experiment that separates them, because guessing between a `SetAlphaData` renormalization and an edit-layer recomposite would have put an unverified cause inside a Critical ticket. Dedup: searched the board for `SetAlphaData` (1 file, the sibling ticket), `weightmap` (8 files, none about destroying other layers), `texelsWithWeight` / `paintedTexels` (2 files), `ULandscapeLayerInfoObject` and `create_procedural_terrain`, and read all fifteen landscape/terrain tickets. Nothing covers weight destruction. NOT attempted: recovery. The level was re-authored around the defect rather than repaired, so whether discarding unsaved state actually restores the other layers is untested, and the ticket does not claim it does.
- `#2-region-is-passed-reach-is-the-recomposite` `IN-REVIEW` developer — Mechanism established from engine source; candidate (1) is refuted and candidate (2) is confirmed as the only region-unbounded step. **The region IS passed to the engine call and IS honored**: `ResolveHeightRegion` (`LandscapeHeightStats.cpp:17-37`) defaults each coordinate independently and clamps, and `SetAlphaData(LayerInfo, PaintMinX, PaintMinY, PaintMaxX, PaintMaxY, …)` receives it. **`SetAlphaData` does not renormalize at all on 5.8**: the single-layer overload (`LandscapeEditInterface.cpp:2106`) derives its component range from X1..Y2, writes only `LayerDataPtrs[UpdateLayerIdx]`, and never touches a sibling channel; its only sibling-facing loop calls `DeleteLayerIfAllZero`, which rescans the whole channel (`:1447`) and so cannot delete a layer that still carries weight. The 10-argument overload (`:2378`) discards `bWeightAdjust`/`bTotalWeightAdjust` as the ticket noted. The handler comment asserting `SetAlphaData renormalizes this layer against the others in the same blend` was therefore **false** and has been corrected: the renormalization is the edit-layer merge's final weight-blending pass (`FLandscapeEditLayersWeightmapsPerformFinalWeightBlendingPS`, `LandscapeEditLayers.cpp:791`). **The reach came from the handler's own completion callback.** `RequestLayersContentUpdateForceAll()` walks every proxy and calls `RequestHeightmapUpdate` + `RequestWeightmapUpdate(bUpdateAll=true, bUpdateCollision=true)` on EVERY component (`LandscapeEditLayers.cpp:6739-6766`), then ORs `Update_All` — a forced heightmap AND weightmap recomposite of the entire landscape in response to a write bounded to `region`, and the only operation in the path whose reach is not the region. It was redundant as well as wide: `SetAlphaData` already calls `Component->RequestWeightmapUpdate()` on exactly the components it wrote (`LandscapeEditInterface.cpp:2229, 2369`), which is what `SettleLandscapeLayers` drains. Narrowed to `RequestLayersContentUpdate(ELandscapeLayerUpdateMode::Update_Weightmap_All)` — the mode the engine itself passes from inside an editing-layer scope after a weightmap edit (`LandscapeEditLayers.cpp:8454`, `LandscapeEdMode.cpp:3470`). **Ticket Fix 1 (census) shipped**: `verify` now samples EVERY `ULandscapeLayerInfoObject` in `LandscapeInfo->Layers` (visibility layer excluded — it is a hole mask, not weight-blended) over the FULL extent before and after the write, and reports `layersAffected[]` (per layer: `painted`, `texelsWithWeightBefore`/`After`, `texelsWithWeightOutsideRegionBefore`/`After`), `otherLayerTexelsLost` (outside the region — non-zero raises a `warnings[]` line), `otherLayerTexelsLostInRegion` (inside — engine-normal at high strength, a number only, no warning), and `censusTexels`. The before half settles first so a preceding `skipFlush` batch is not misattributed to this paint; the census is capped at 4,194,304 texels per pass and says so in `warnings[]` when skipped, with the fields absent rather than zero. **Ticket Fix 2 (transaction) shipped**: the whole write is inside one `FScopedTransaction("Paint Landscape Layer")` opened before `MarkLevelActorModified` and closed before the settle, so `editor.undo` reverses it. **Contract fix**: the verb summary and `Docs/wiki-src/landscape.md` now state that painting a weight-blended layer costs the siblings their weight INSIDE the painted region — engine behaviour, the same thing `FLandscapeToolStrokePaint::Apply` does by writing every member of the target layer's blend group — so multi-layer terrain is built by painting each layer over the region it should own, not by painting one over the full extent and another on top. Regression test `PinWright.landscape.create_procedural_terrain.RegionPaintLeavesOtherLayersOutsideRegion` (`Tests/Environment/TestLandscapePaintLayerHonesty.cpp`): 2x1-component landscape, two target layers, sibling painted over the full extent, second layer painted over a region derived from the live extent, then asserts by direct `GetWeightDataFast` that the painted layer gained weight inside the region and the sibling kept **every** weighted texel outside it, plus that the response's own census reaches the same verdict (`layersAffected[]` present, `otherLayerTexelsLost == 0`). **Two honest limitations.** (a) NOT COMPILED AND NOT RUN — this wave builds after the fact, so "fails before the fix" is unverified for the behavioral half; the census half provably fails before (the fields did not exist). (b) The reporter's landscape-wide observation was NOT reproduced here, and source reading alone does not explain it: with the write region- and layer-bounded and the merge normalizing per texel, a sibling at 255 outside the region normalizes back to 255. The narrowed callback removes the only unbounded step and the census now measures the outcome in-band on every call, which is what the ticket's own experiment was for — if the loss survives, `otherLayerTexelsLost` will name it instead of `texelsWithWeight == paintedTexels` reporting clean over it.
- `#3-layerinfo-unbinding-is-the-mechanism` `OPEN` developer — **Reading (B) held: the sibling really was destroyed.** `PinWright.landscape.create_procedural_terrain.RegionPaintLeavesOtherLayersOutsideRegion` failed on `sibling layer weight is readable after the region paint` and `census covers both target layers (got 1)`; the *before* halves of both passed (the fixture's sibling was created, painted, and carried weight outside the region), so the fixture is sound and the after-state is the damage. The census found one layer because the paint had already **unbound** the other, not because the fixture never made it. **Mechanism — found, and it is neither of #2's candidates.** The handler creates the `ULandscapeLayerInfoObject` and binds it ONLY into `ULandscapeInfo::Layers`. From 5.5 on, `UpdateLayerInfoMapInternal` does `Swap(Layers, PreviousLayers)` and repopulates `Layers` **purely from `LandscapeActor->GetTargetLayers()`** (`Landscape.cpp:4582-4595`); it no longer harvests component weightmap allocations the way 5.3/5.4 did (`Landscape.cpp:3936-3987` on 5.4), which is why this is a 5.5-5.8 regression and why nothing in the 5.3/5.4 reading predicted it. Assigning a landscape material does **not** put its declared names into `TargetLayers` (only the load-time `TargetLayersForFixup` deprecation path writes there), so the auto-created binding lived nowhere persistent — and the handler itself calls `LandscapeInfo->UpdateLayerInfoMap()` at the top of EVERY paint, so the first thing paint N+1 does is unbind every layer paints 1..N created. **Then the erase:** `ALandscape::GetTargetLayerNames` transforms `LandscapeInfo->Layers` filtered on `LayerInfoObj != nullptr` (`LandscapeEdit.cpp:8323-8334`); `PerformLayersWeightmapsBatchedMerge` takes exactly that as `RequestedWeightmapLayerNames` (`LandscapeEditLayers.cpp:5883`); and `ReallocateLayersWeightmaps` deletes from every resolved component every `FWeightmapLayerAllocationInfo` whose `LayerInfo` is not in the merge's per-component list (`:5744-5755`). The unbound sibling's allocation is removed from every component of every proxy — the whole landscape's weight for that layer, gone, **independent of `region` and of `strength`**. That is the reporter's #1 observation reproduced exactly, including the half #2 could not explain: *every layer other than the one named*, over the *full extent*, because only the layer named in the current call gets its binding re-created. **#2 is partly refuted.** Its findings that the region is honored, that `SetAlphaData` does not renormalize, and that `RequestLayersContentUpdateForceAll` was redundant and wide all stand; its closing claim that narrowing the callback removed "the only unbounded step" does not — the unbounded step was the unbinding, which #2 never looked at. **Why Fix 1's census stayed blind:** it enumerates `LandscapeInfo->Layers` and skips entries with no `LayerInfoObj`, and the unbinding happens BEFORE the "before" half runs, so the sibling is invisible to both halves and `otherLayerTexelsLost` reports 0 over the destruction. **Fix shipped, NOT COMPILED AND NOT RUN:** the auto-create branch now registers the LayerInfo where it persists — `LandscapeInfo->CreateTargetLayerSettingsFor(NewLayerInfo)` on 5.5+, `CreateLayerEditorSettingsFor` on 5.3/5.4 — before touching `Layers`, then re-scans `Layers` by name because the registration's `PostEditChangeProperty` can rebuild it and invalidate the index taken earlier. That is the engine's own idiom: `ULandscapeInfo::CreateTargetLayerSettingsFor` (`Landscape.cpp:4478`, Update- or AddTargetLayer per proxy) is what the Target Layers panel calls (`LandscapeEditorDetailCustomization_TargetLayers.cpp:2371`), and `ALandscapeProxy::Import` registers through the same API immediately after its own `SetAlphaData` (`LandscapeEdit.cpp:3797-3800`). Side effect: repeat paints of one layer stop manufacturing a second `LayerInfo_<Name>` in the same outer, so `layerInfoAutoCreated` is true only on the first paint of a layer. Also corrected two now-false comments that named the wrong cause — the handler's "the only step in this path whose reach is not the region" and the test's counterfactual paragraph. **Status returned to `OPEN`, not `IN-REVIEW`:** the fix is unbuilt and unrun, and shipping-unverified is what produced this cycle. It earns `IN-REVIEW` when that named test runs green. **Severity: Critical stands**, and the reach case is stronger than #1 argued — the defect fires on the second paint of a second layer, which is the first call that can reveal it, on every 5.5-5.8 host. **Remaining hazard, deliberately NOT fixed here:** a layer already in the orphaned state (weight allocated on components, no `LayerInfoObj` in `Layers`) is still erased silently by the next paint, and the census still cannot see it for the same reason. A detector would walk each component's `GetWeightmapLayerAllocations()` for `LayerInfo` objects absent from `LandscapeInfo->Layers` and warn before the write; that is the next honest extension of Fix 1. Files: `Handlers/Environment/LandscapeHandler.cpp` (registration + comment correction), `Tests/Environment/TestLandscapePaintLayerHonesty.cpp` (counterfactual comment only — no assertion changed).
