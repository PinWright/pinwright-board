---
id: B-paint-blanks-unallocated-layers-component-wide
title: "landscape.create_procedural_terrain's first paint on a virgin component blanks every OTHER material-declared layer across that WHOLE component — 6300 uu here, not the 51x51-texel region asked for — and otherLayerTexelsLost correctly reports 0, because the loss is an allocation-side change no texel census can express"
status: IN-REVIEW
severity: High
category: bug
tags: [landscape, create_procedural_terrain, weightmap, layer-paint, weightmap-allocation, per-component, landscape-grass, census-blind, silent-wrong-output, material-permutation, blast-radius]
encounters: 1
lastSeen: 2026-08-30T17:00:00+05:00
---

# The blast radius is the component, not the region, and the verb's two honesty instruments are both correct and both silent

Painting `Rock` at strength 1.0 over a **51x51-texel** rectangle wiped the landscape grass across the
**whole visible frame**. `otherLayerTexelsLost` reported `0`. The orphaned-allocation scan reported
nothing. Both were **truthful**. Nothing was corrupted, nothing was lost on disk, and the response was
`success: true` with a clean census over a frame that had lost its groundcover.

This is not a wrong number. It is a **missing** number: the quantity that decides the outcome —
*which components gained their first weightmap allocation, and which registered layers those components
do not carry one for* — is never computed, even though the walk that would compute it is already running
in the same call.

## Measured, on the 13:32 build (plugin `d8f1bc32`, waves 2 and 3), live, this session

Source of record: `Docs/map/vegetation-test-level.md` §
*"CORRECTION: the landscape carries NO painted layer weight, and multi-layer painting was tried and
reverted"* (project commit `7d629ad9`). All rows at pose `(-1000, 11000, 1106)` pitch -6 yaw 225,
1280x720, exposure pinned `ev100 -0.5`.

| step | mean lum | bytes | frame |
|---|---|---|---|
| baseline | 0.4890 | 2,425,329 | full grass carpet over the scree |
| `Rock` 1.0 over px (192,342)-(242,392) | 0.6073 | 1,714,497 | **grass gone, ground flat pale cream** |
| `Grass` 1.0 full extent | 0.5121 | 2,383,347 | carpet fully restored |
| `Grass` 0 + `Rock` 1.0 over that region | 0.5876 | 1,963,700 | grass gone **inside the rectangle only**, hard straight edge |
| `Rock` 0 + `Grass` 1.0 over that region | 0.4916 | 2,420,511 | back to baseline |

**Row 2 against row 4 is the whole proof.** The same `Rock` paint, the same rectangle, two different
blast radii. The only thing that changed between them is whether `Grass` already carried registered
weight — and therefore an allocation — on the components under that rectangle. With no `Grass`
allocation, `Rock`'s arrival takes the grass out to the component boundary. With one, the loss is
confined to the requested texels, with the hard straight edge a rectangle should produce.

**Row 5 is why this is not a Critical.** The damage reverses completely: 0.4916 against a 0.4890
baseline is inside the noise of two consecutive captures, and the revert was confirmed at a second pose
28 km away (0.5975/0.5972 -> 0.5977). The weightmaps on disk were never touched — the landscape loads
from disk with empty weightmaps, established separately in the same section — and
`Content/Maps/PW_VegetationTest.umap` is byte-unchanged.

Landscape context, from the same calls: `orphanScanComponents 64`, `orphanScanProxies 1`,
`censusTexels 255025`. `otherLayerTexelsLost` was `0` on **all six** paints of that pass and
`LANDSCAPE_ORPHANED_LAYER_WEIGHT` never fired.

## Why `otherLayerTexelsLost` cannot move — re-derived at HEAD `1a9e5778`

`Handlers/Environment/LandscapeHandler.cpp` is clean against `HEAD`; every line below is a HEAD line.

`otherLayerTexelsLost` is a **difference of two counts of non-zero weightmap bytes for one layer**:

```cpp
OtherLayerTexelsLost += FMath::Max(0, Entry.OutsideBefore - Entry.OutsideAfter);   // :3074
```

`OutsideBefore` / `OutsideAfter` come from `SampleLayerWeights` (`:2893`), whose only observable is
`FLandscapeEditDataInterface::GetWeightDataFast(Info, FullMinX, ...)` (`:2904-2905`), tallied at
`if (Weights[X + Y * SizeX] == 0) { continue; }` (`:2908`) and split by the region test (`:2911-2916`).
The census set is `LandscapeInfo->Layers` minus null layer infos and the visibility layer
(`:2927-2930`); the field is published at `:3139`.

Three independent reasons a per-component allocation change cannot move that number:

1. **It counts bytes, and the paint wrote none of `Grass`'s.** Painting `Rock` writes `Rock` channels.
   `Grass` had 0 texels carrying weight before and 0 after, so `max(0, 0 - 0) = 0`. Arithmetically
   correct. There is no bug in the metric.
2. **`GetWeightDataFast` has one symbol for two states.** A texel of a layer the component has **no
   allocation for** and a texel of an **allocated** layer whose weight is 0 both read back as byte 0.
   The census's alphabet is strictly smaller than the render path's, so the distinction that decides
   the frame is not representable in it at all.
3. **Nothing was lost, so no loss metric can fire.** The damage is a change to *which shader
   permutation the component compiles*, caused by the **other** layer gaining an allocation. There is
   no texel count anywhere that moves. A perfect, orphan-aware, all-layers texel census — which is
   exactly what waves 2/3 built — is still structurally blind to it.

**The orphan scan is blind for a second, different reason, and it is worth stating separately** because
it is the verb's only allocation-side read and a reader will assume it covers this.
`ScanLandscapeOrphanedWeightAllocations` (`:2297`, called unconditionally at `:2710`) *does* walk every
component's `GetWeightmapLayerAllocations(false)` (`:2343-2344`) — the walk this ticket needs is already
paid for — but its predicate skips any allocation whose `LayerInfo` is in the registered set
(`:2359-2361`, set built at `:2309-2313`). `Rock` is a registered target layer. A component gaining a
`Rock` allocation is therefore, by construction, not an orphan and not reported. Both instruments are
correct; they answer two questions, and neither is this one.

**Those are the only allocation-side reads in the plugin.** `GetWeightmapLayerAllocations`,
`WeightmapLayerAllocations` and `FWeightmapLayerAllocationInfo` occur at exactly
`LandscapeHandler.cpp:2343-2346` and `Tests/Environment/TestLandscapePaintLayerHonesty.cpp:898-900`
across all of `Source/`. Nothing else in PinWright has ever looked at an allocation array.

**The in-band docs point the caller at the wrong gap.** The registered `verify` description (`:2471`)
and `Docs/wiki-src/landscape.md:252` both now disclose the *orphan* limit of the census — "weight
allocated for a layer that is NOT a registered target layer is invisible to it" — and both stop there,
which reads as "with the orphan scan running, `otherLayerTexelsLost: 0` is trustworthy". The handler
summary (`:2463`) goes further and sells the census as the reason to trust a multi-layer paint: *"the
cost to the other layers is a reported number instead of a discovery in Landscape Ed Mode."* On this
class of damage the cost is not a reported number and the discovery is in the viewport.

## The engine mechanism: what is sourced, and what is only measured

**Sourced (5.8, `C:/UE_5.8/Engine/Source/Runtime/`).** Weightmap allocation is per
`ULandscapeComponent` — `TArray<FWeightmapLayerAllocationInfo> WeightmapLayerAllocations`
(`Landscape/Classes/LandscapeComponent.h:576`), reached through four `LANDSCAPE_API` overloads at
`:819-822`. The shader permutation a component renders with is **keyed by, and built from, that
component's allocation list**: `ULandscapeComponent::GetLayerAllocationKey`
(`Landscape/Private/LandscapeEdit.cpp:549`) hashes the allocations into the MIC key, and
`GetCombinationMaterial` (`:583`) adds one `FStaticTerrainLayerWeightParameter` per allocation and
nothing else —

```cpp
for (const FWeightmapLayerAllocationInfo& Allocation : Allocations)          // :638
  if (Allocation.LayerInfo)
    StaticParameters.EditorOnly.TerrainLayerWeightParameters.Add(
        FStaticTerrainLayerWeightParameter(LayerName, Allocation.WeightmapTextureIndex));  // :645
```

— called with that component's `WeightmapBaseLayerAllocation` at `:710`. A layer **absent** from that
per-component set gets no match in `FHLSLMaterialTranslator::StaticTerrainLayerWeight`
(`Engine/Private/Materials/HLSLMaterialTranslator.cpp:9030`), which returns `INDEX_NONE` (`:9089-9101`),
and `UMaterialExpressionLandscapeLayerSample::Compile`
(`Landscape/Private/Materials/MaterialExpressionLandscapeLayerSample.cpp:38-50`) turns that into
`Compiler->Constant(0.f)` under the engine's own comment: *"layer is not used in this component, sample
value is 0."* The single escape to the node's `PreviewWeight` is
`if ((NumWeightmapParameters == 0) && Material->IsPreview())` (`:9089-9091`), and `IsPreview()` is
`false` by default (`Engine/Public/MaterialShared.h:2592`) with overrides returning `true` only in
material-editor code (`Editor/UnrealEd/Private/PreviewMaterial.cpp:245`,
`Editor/MaterialEditor/Private/MaterialEditor.h:123`, `Editor/MaterialEditor/Public/MaterialStatsCommon.h:29`)
— never at level render.

That is the half this ticket needs, and it is sourced: **a component that gains an allocation for one
layer makes every other material-declared layer it does not carry an allocation for compile to a
constant 0 across all of it.** The map's `LandscapeGrassOutput` is fed by exactly such a node —
`LandscapeLayerSample("Grass")` (`Docs/map/vegetation-test-level.md:81-85`) — so grass density goes to
zero over the whole component.

**NOT sourced, and stated rather than papered over.** Why the *virgin* component — zero allocations for
anything — renders grass **at all**. A plain reading of `:9089-9091` predicts `Constant(0.f)` there too,
since a level render is never preview and `GetCombinationMaterial` has no early-out for an empty
allocation list. Measured behaviour is the opposite: on this landscape the carpet renders at full
density with the weightmaps empty, and painting `Grass` 1.0 over the full extent moved mean luminance by
`+0.0011`, inside two-capture noise. **That direction is empirical only.** The candidate explanation I
found but did **not** close: `ULandscapeComponent::GrassData` is a *serialized* per-component cache
(`Landscape/Private/Landscape.cpp:1092-1119`) and runtime regeneration is off by default
(`GGrassMapUseRuntimeGeneration = 0`, `Landscape/Private/LandscapeGrassMapsBuilder.cpp:43-46`), so the
carpet on a virgin component may be a cached grass map rather than a live sample, dropped the moment the
component's weightmap changes. I did not trace the invalidation and do not assert it.

**Consequence for a fixer, and a correction to the project doc.**
`Docs/map/vegetation-test-level.md:297-302` states that on a component with no weightmap allocation the
`Grass` sample *"does not resolve to zero."* That sentence is **not source-confirmed and a plain reading
of the translator contradicts it**; it should be read as a record of what was observed, not as a
mechanism. Nothing in this ticket rests on it: the finding needs only the sourced half above plus the
row-2-against-row-4 A/B, and both are intact either way.

**One inference, flagged as such.** In row 4, `Grass` was painted to strength 0 over the region before
`Rock`. Writing a zero byte into an allocated channel does not remove the allocation, so `Grass` stayed
allocated on those components and its sample stayed a real texture read — 0 inside the rectangle, 1.0
outside — which is why that frame has a hard straight edge instead of a component-sized hole. Consistent
with every row, but derived, not separately measured.

## The component is 6300 uu here, and that is derivable

`Docs/map/vegetation-test-level.md:13-15`: landscape scale `(100, 100, 100)`, `8 x 8` components,
`ComponentSizeQuads 63`, `NumSubsections 1`, 505 x 505 heightmap vertices. One component is
`63 quads x 100 uu = 6300 uu`. Every reported number cross-checks: `8 x 63 + 1 = 505` vertices,
`505^2 = 255025` = the reported `censusTexels`, `8 x 8 = 64` = the reported `orphanScanComponents`.
A requested 51x51-texel rectangle is 5100 uu and spans up to 2x2 components, i.e. up to
**12600 x 12600 uu** of actual effect — 6.1x the requested area, and enough to fill the near and mid
ground of the measured frame.

Nothing in the response says how large a component is, so a caller cannot compute this even after
being told it matters.

## Ask

1. **A per-component, allocation-side term in the verify block.** The walk is already running
   (`:2343-2344`), so the cost is a second predicate on an existing loop, not a second traversal.
   Record per component the registered layer infos it holds an allocation for, before and after, and
   publish:
   - `componentsGainingAllocation` — components whose base allocation array grew, and
     `componentsGainingFirstAllocation` — the subset that went from empty to non-empty, which is the
     expensive transition;
   - `layersNowSampledZeroOnTouchedComponents[]` — for each component the paint allocated on, the
     registered target layers that component does **not** carry an allocation for. Those are exactly
     the layers whose `LandscapeLayerSample` / `LandscapeLayerWeight` compiles to `Constant(0.f)` there
     (`MaterialExpressionLandscapeLayerSample.cpp:43-46`);
   - `componentSizeUU` and the component footprint the paint actually touched, so the blast radius is a
     number rather than something the caller has to derive from `ComponentSizeQuads` times `DrawScale`.
2. **At minimum, a `warnings[]` line** when a paint creates the **first** allocation on any component,
   naming the sibling layers that go to zero across it and the world-space footprint. This is the
   cheapest thing that would have prevented the measured surprise.
3. **Stop the census's disclosure at `:2471` and `Docs/wiki-src/landscape.md:252` reading as a
   guarantee.** Both currently name the orphan gap and stop, which implies the orphan scan closes the
   census. Say plainly that `otherLayerTexelsLost` is a texel-count difference and cannot express a
   loss where no texel changed, and that a clean census does not mean the frame is unchanged.
4. **The doc line that tells callers the order of work.** The first paint on a virgin component is the
   expensive one; paint the base layer over the full extent **first**, then accents over their regions
   (plus the base to 0 there). `Docs/map/vegetation-test-level.md:357-361` reached the same order
   empirically for this map — it belongs in `Docs/wiki-src/landscape.md` as a general rule, because it
   is a property of the engine's per-component permutation, not of this map.

**Not asked for: a refusal.** Unlike `B-paint-erases-orphaned-layer`, nothing here is destroyed and
everything reverses, so refusing would obstruct the normal way multi-layer terrain gets built. Report
and warn is the right weight.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as* — *the call succeeds, every number it
reports is correct, and the output is wrong because the deciding number was never reported.* This is
a member, and an unusually pure one: the reported number is not merely correct but **provably** correct
(`Grass` held no weight, so it could lose none), while the deciding number is never computed at all,
despite the loop that would compute it already running two hundred lines earlier in the same call.

## Distinct from

- **`B-paint-erases-orphaned-layer`** (IN-REVIEW, High) — the closest neighbour, and it shares the
  phrase "the census cannot see". Different defect. There, an **already-orphaned** layer's weight is
  really erased, landscape-wide, irreversibly, and the census misses it because it enumerates the
  registration side. Here **nothing is erased**: every layer is registered, no weightmap byte changes,
  the census is right, the orphan scan is right, and the frame still loses its groundcover because the
  *other* layer gained an allocation. One is a loss the census cannot see; this is damage that is not a
  loss at all. Its fix — the allocation-side scan it added — is the right place to land this one's
  counters, which is a reason to sequence them, not to merge them.
- **`B-paint-layer-destroys-other-layer-weights`** (IN-REVIEW, Critical) — claims painting one layer
  zeroes every OTHER layer's weight landscape-wide. **Deliberately not merged, and deliberately not
  edited.** The same measurement pass that produced this ticket's table recorded that its premise *does
  not reproduce on this build*: across six paints `otherLayerTexelsLost` was 0 every time and every loss
  was confined to the requested rectangle. That contradiction belongs to whoever is re-deriving that
  ticket. It is also worth their attention that the *visible* symptom this ticket documents — grass
  gone far outside the region — is the symptom that would most easily be mistaken for that ticket's
  claim, by an observer looking at the frame rather than at the weightmaps.
- **`F-landscape-paint-region-shapes`** (OPEN, Medium) — the region is an axis-aligned rectangle with
  no brush or mask. Adjacent and independent: that ticket is about the shape the caller can *request*,
  this one about the footprint the engine actually *changes*. Both were reasons the multi-layer attempt
  was reverted; neither fixes the other.
- **`B-create-procedural-terrain-paints-nothing`** (DONE, High) — introduced the single-layer readback.
  Untouched: it never claimed to look outside the requested layer, let alone at allocations.
- **`B-game-view-suppresses-landscape-grass`** (OPEN, Medium) and
  **`B-ortho-capture-renders-no-landscape-grass`** (IN-REVIEW, High) — both are grass missing from a
  *capture* of an unchanged world. This one is grass missing from the world itself; a second capture
  path would show the same absence. Named because "landscape grass is missing" dedups onto them.
- **`B-compile-material-landscape-consumers-stale`** (DONE, High) — combination-MIC invalidation after a
  master-graph edit. Same MIC, opposite direction: there the permutation failed to update; here it
  updated correctly and the correct update is the damage.

## Not done

No source was modified and no test was written. The editor (pid 18592) was not touched: this is a
filing pass over already-measured data plus a source re-derivation. Every plugin citation is HEAD
`1a9e5778`; the measurements are the **13:32 build**, plugin `d8f1bc32`, which HEAD leads. The proposed
counters have not been prototyped, and no landscape other than this one was surveyed — the 6300 uu
figure is this map's, the mechanism is not.

severity rationale: impact=High — silent wrong output on a normal path, and the specific form the README names, "the caller trusts a result that is a lie and builds on it": the verb returns `success: true` with `otherLayerTexelsLost: 0`, an empty orphan report and a registered summary promising that "the cost to the other layers is a reported number instead of a discovery in Landscape Ed Mode" (`LandscapeHandler.cpp:2463`), over a frame that has lost its groundcover across 6.1x the requested area; the lie is not in the number, which is provably correct, but in the verification contract those numbers are sold under, and there is no other field a caller could read to find out × reach=normal — **the reach bump-down is declined**, and this is the specific reading being rejected: `landscape.create_procedural_terrain` is an infrequent verb, which argues down one to Medium, but the rubric's modifier asks whether the *affected path* is a rare edge path, and this one is the opposite — it fires on the FIRST paint of any landscape whose weightmaps are empty, which is the state every freshly created landscape is in, so every caller building multi-layer terrain hits it on call one and stops hitting it once the base layer is down; how often the verb itself is reached is an `encounters` signal, which the README excludes as a severity input. Critical is declined on the band's own words — "a write that corrupts or loses asset data": no weightmap byte changed, the map file is byte-unchanged, and the damage reverses exactly (row 5, 0.4916 against a 0.4890 baseline, inside two-capture noise, confirmed at a second pose 28 km away), so rating it Critical would order it above `B-paint-erases-orphaned-layer`'s genuinely irreversible landscape-wide erase, which is the wrong order for the picker. Medium is declined because Medium is a soft blocker with a documented workaround, and there is none: the per-component rule appears nowhere in `Docs/wiki-src/landscape.md`, and the instrument a caller would use to check — the verb's own census — is precisely the thing that certifies clean over it -> High

## History
- `#1-component-wide-blanking-census-cannot-express` `OPEN` reporter — Filed from live measurement on the **13:32 build** (plugin `d8f1bc32`) plus a source re-derivation at HEAD `1a9e5778`; the editor was not touched for this pass. **The measurement.** Painting `Rock` 1.0 over a 51x51-texel rectangle removed the landscape grass from the whole visible frame (mean lum 0.4890 -> 0.6073, 2,425,329 -> 1,714,497 bytes); painting `Grass` 1.0 over the full extent restored it (0.5121); repeating the identical `Rock` paint *after* `Grass` carried weight confined the loss to the rectangle with a hard straight edge (0.5876); reverting returned the frame to baseline (0.4916 vs 0.4890, inside two-capture noise, confirmed at a second pose 28 km away). `otherLayerTexelsLost` was `0` on all six paints and `LANDSCAPE_ORPHANED_LAYER_WEIGHT` never fired. Full table and provenance in `Docs/map/vegetation-test-level.md` § *CORRECTION: the landscape carries NO painted layer weight...* (project commit `7d629ad9`). **Row 2 against row 4 is the proof and needs no mechanism**: identical paint, two blast radii, one difference — whether the sibling layer already held an allocation on those components. **Why the census cannot move, re-derived rather than inherited.** `otherLayerTexelsLost` is `max(0, OutsideBefore - OutsideAfter)` summed over registered non-requested layers (`LandscapeHandler.cpp:3074`, published `:3139`), and its only observable is `GetWeightDataFast` counting non-zero bytes over the full extent (`:2893`, `:2904-2905`, `:2908`, `:2911-2916`) across `LandscapeInfo->Layers` minus nulls and the visibility layer (`:2927-2930`). Three reasons it reads 0 truthfully: the paint wrote no `Grass` byte so both counts are 0; `GetWeightDataFast` returns byte 0 for BOTH "no allocation" and "allocated, weight 0", so the deciding state is not representable in the census's alphabet; and nothing was lost, so no loss metric can fire — the change is to which shader permutation the component compiles, on the other layer's side. **The orphan scan is blind for a second and different reason**, worth separating because it is the plugin's only allocation-side read and looks like it should cover this: `ScanLandscapeOrphanedWeightAllocations` (`:2297`, called at `:2710`) does walk every component's `GetWeightmapLayerAllocations(false)` (`:2343-2344`) — so the walk this ticket needs is already paid for — but skips any allocation whose `LayerInfo` is registered (`:2359-2361`, set at `:2309-2313`), and `Rock` is registered. Confirmed those are the only allocation-side reads anywhere in `Source/`: `GetWeightmapLayerAllocations` / `WeightmapLayerAllocations` / `FWeightmapLayerAllocationInfo` occur at `LandscapeHandler.cpp:2343-2346` and `TestLandscapePaintLayerHonesty.cpp:898-900` and nowhere else. **Engine mechanism, split honestly into sourced and empirical because the brief for this filing demanded it and the split turned out to matter.** SOURCED on 5.8: allocation is per `ULandscapeComponent` (`LandscapeComponent.h:576`, accessors `:819-822`); the MIC permutation is keyed by that component's allocation list (`GetLayerAllocationKey`, `LandscapeEdit.cpp:549`) and built one `FStaticTerrainLayerWeightParameter` per allocation and nothing else (`:638-648`, called with `WeightmapBaseLayerAllocation` at `:710`); a layer absent from that set gets `INDEX_NONE` from `FHLSLMaterialTranslator::StaticTerrainLayerWeight` (`HLSLMaterialTranslator.cpp:9030`, `:9089-9101`) and `UMaterialExpressionLandscapeLayerSample::Compile` turns that into `Compiler->Constant(0.f)` under the engine's own comment *"layer is not used in this component, sample value is 0"* (`MaterialExpressionLandscapeLayerSample.cpp:39-51`); the only escape to `PreviewWeight` needs `NumWeightmapParameters == 0 && Material->IsPreview()` (`:9089-9091`) and `IsPreview()` is false by default (`MaterialShared.h:2592`) with `true` only in material-editor code (`PreviewMaterial.cpp:245`, `MaterialEditor.h:123`, `MaterialStatsCommon.h:29`). That establishes the half the ticket rests on: gaining an allocation for one layer is what drives an un-allocated sibling to constant zero across the whole component, and the map's grass output is fed by exactly such a node (`vegetation-test-level.md:81-85`). **NOT SOURCED, and recorded as a gap rather than guessed at:** why a virgin zero-allocation component renders grass at all. A plain reading of `:9089-9091` predicts `Constant(0.f)` there too — a level render is never preview, and `GetCombinationMaterial` has no early-out for an empty allocation list — which is the opposite of what was measured. Candidate found but NOT closed: `ULandscapeComponent::GrassData` is a serialized per-component cache (`Landscape.cpp:1092-1119`) with runtime regeneration off by default (`GGrassMapUseRuntimeGeneration = 0`, `LandscapeGrassMapsBuilder.cpp:43-46`), so the virgin carpet may be a cached grass map rather than a live sample; the invalidation was not traced and is not asserted. **A relayed premise did NOT survive and is corrected here:** `Docs/map/vegetation-test-level.md:297-302` states that on a component with no allocation the `Grass` sample "does not resolve to zero" — that is an observation stated as a mechanism, and the translator contradicts it. The ticket was rewritten to rest on the sourced half plus the A/B instead, which it does without loss. **Component size confirmed derivable, not taken on trust:** scale `(100,100,100)`, `ComponentSizeQuads 63`, 8x8 components (`vegetation-test-level.md:13-15`) gives 6300 uu, and every reported number cross-checks — `8*63+1 = 505`, `505^2 = 255025` = reported `censusTexels`, `8*8 = 64` = reported `orphanScanComponents`. A 51x51-texel request spans up to 2x2 components, i.e. up to 12600 uu square: 6.1x the requested area. Nothing in the response publishes component size, so a caller cannot derive the blast radius even after being told it matters. **Dedup.** Grepped the board for `create_procedural_terrain` (20 files), `otherLayerTexelsLost` (2), `weightmap` (11), and landscape-grass terms (8), and read the frontmatter of every hit plus `B-paint-erases-orphaned-layer` in full. Nothing owns this. The closest, `B-paint-erases-orphaned-layer`, shares the phrase "census cannot see" and is a genuinely different defect — there weight IS erased and the census misses it; here no weight moves, the census is right, the orphan scan is right, and the frame changes anyway — but its allocation-side scan is the correct landing site for this ticket's counters, so they should be sequenced rather than merged. `B-paint-layer-destroys-other-layer-weights` (Critical, IN-REVIEW) is deliberately NOT edited and NOT merged: the same measurement pass recorded that its premise does not reproduce on this build, which is that ticket's own re-derivation to make, and the visible symptom documented here is the one most likely to be mistaken for its claim by an observer reading the frame instead of the weightmaps. **Severity called High**, with both alternatives argued down in the rationale line: not Critical because no byte on disk changed and the damage reverses exactly, and rating it above an irreversible erase would mis-order the picker; not Medium because there is no documented workaround and the instrument a caller would check with is the thing that certifies clean. The reach bump-down is explicitly declined: the verb is infrequent, but the defect sits on its most common first call rather than on an edge path, and verb popularity is an `encounters` signal the README excludes from severity. **Not done:** no source modified, no test written, no editor call made, no prototype of the proposed counters, and no landscape other than this one surveyed.
- `#2-cross-link-third-collapse-on-the-same-read` `OPEN` reporter — **Cross-link only. This is not a re-observation of this ticket's defect, and `encounters` / `lastSeen` are deliberately left untouched** — bumping a work-ordering tiebreak or the last-seen date would tell a picker the component-wide blanking was seen again, which it was not. (Same reasoning `B-paint-layer-destroys-other-layer-weights` `#5` gave for the same choice.) **A third state collapse on the same read has been filed as `B-paint-census-counts-texels-not-weight` (OPEN, Medium).** `#1` establishes two: `GetWeightDataFast` returns byte 0 for **both** "the component has no allocation for this layer" and "allocated, weight 0", so a per-component allocation change is not representable in the census's alphabet at all. The third is magnitude — `SampleLayerWeights` tallies `if (Weights[X + Y * SizeX] == 0) { continue; }` (`LandscapeHandler.cpp:2908`, re-derived at HEAD `1a9e5778` for that filing), so everything from 1 to 255 is one symbol and a sibling renormalised from 255 down to 1 contributes 0 to both `otherLayerTexelsLost` (`:3074`) and `otherLayerTexelsLostInRegion` (`:3075`). **Filed as a separate ticket rather than an encounter here, on this ticket's own argument.** The tempting merge is "both are 'the census counts presence, not the thing that matters', one seat" — but `#1` already answers it: *"A perfect, orphan-aware, all-layers texel census — which is exactly what waves 2/3 built — is still structurally blind to it."* Concretely, neither fix closes the other. Summing weight magnitude at `:2908` reveals nothing about which components gained an allocation: that lives in `GetWeightmapLayerAllocations`, walked at `:2343-2344` in a different function ~550 lines away, and is not a weightmap read at all. Conversely, landing this ticket's `componentsGainingAllocation` / `componentsGainingFirstAllocation` / `layersNowSampledZeroOnTouchedComponents[]` leaves `otherLayerTexelsLost` exactly as insensitive to 255 -> 1 as it is now. Two data sources, two seats, two independently landable fixes — and two bands, since that one is Medium (see below) against this ticket's High, so bundling would force one severity onto items the picker should order separately, the same objection `B-layer-paint-doc-claims-blend-group-write` § *Structure* raises. **Sequencing recorded in both places: take this ticket first.** It needs a new walk-and-diff and a new response shape; the magnitude term is a second accumulator inside a loop that already runs over the same buffer, so it lands cleanly on top of this ticket's work rather than the other way round. `#1`'s § *Distinct from* already says this ticket's allocation-side scan is "the right place to land this one's counters" about `B-paint-erases-orphaned-layer`; the magnitude counters want the same seat, and all three should be shaped as one `layersAffected[]` extension rather than three response revisions. **Why the magnitude ticket rates a band below this one, stated so the ordering is not re-litigated:** it is currently **latent**. Nothing on this verb's default path is weight-blended — every auto-created `ULandscapeLayerInfoObject` lands on `ELandscapeTargetLayerBlendMethod::None` (`LandscapeLayerInfoObject.cpp:27` -> `LandscapeSettings.h:167`, unset in this project's `Config/`, its `Saved/Config/` and `C:/UE_5.8/Engine/Config/`; the merge's final weight-blending pass is gated on `FinalWeightBlending` at `LandscapeEditLayers.cpp:791`/`:3594`), which is filed as `B-paint-auto-created-layer-never-weight-blended` (OPEN, High). So no renormalisation happens and `otherLayerTexelsLostInRegion: 0` is presently the *true* answer, not a lie — a precision gap rather than silent wrong data. **This ticket's damage, by contrast, is visible in the frame today**, which is exactly why it is High and the magnitude ticket is not, and why rating the latent one High pre-emptively would have mis-ordered the picker against this one. It escalates to High when the blend-method ticket reaches DONE. **Nothing in this ticket is corrected or contradicted by any of that.** `#1`'s three reasons `otherLayerTexelsLost` cannot move are unaffected: reason 1 (the paint wrote no `Grass` byte, so `max(0, 0 - 0) = 0`) is arithmetic; reason 2 is the allocation/zero collapse, which the magnitude finding sits beside rather than on top of; reason 3 (nothing was lost, so no loss metric can fire) is untouched. The row-2-against-row-4 A/B, the sourced permutation mechanism (`LandscapeComponent.h:576`, `LandscapeEdit.cpp:549`/`:638-648`/`:710`, `HLSLMaterialTranslator.cpp:9030`/`:9089-9101`, `MaterialExpressionLandscapeLayerSample.cpp:39-51`) and the 6300 uu derivation all stand as written. **Not done for this entry:** no source read beyond the four `LandscapeHandler.cpp` lines cited above, no measurement, no editor call (pid 18592 untouched), and this ticket's body prose above `## History` was not edited.
- `#3-per-component-allocation-blast-radius` `IN-REVIEW` developer — Implemented asks 1–4 in `Handlers/Environment/LandscapeHandler.cpp` and `Docs/wiki-src/landscape.md`. **Reproduction re-verified in THIS checkout before writing code, structurally:** `grep -rn "GetWeightmapLayerAllocations\|WeightmapLayerAllocations\|FWeightmapLayerAllocationInfo" Source/` still returned exactly the orphan scan plus `TestLandscapePaintLayerHonesty.cpp` — no per-component before/after diff existed anywhere, and the response carried no allocation-side field beyond the orphan counters, so `otherLayerTexelsLost` (still `max(0, OutsideBefore - OutsideAfter)` over non-zero bytes) was the only thing a caller could read. **The measurement.** New file-static `ScanLandscapeComponentAllocationCoverage` walks `ForEachLandscapeProxy` -> `LandscapeComponents` -> `GetWeightmapLayerAllocations(false)` and records, per component, the set of layer NAMES it holds an `IsAllocated()` channel for. It `FindOrAdd`s an entry even for a component with nothing allocated, because "present with an empty set" is what distinguishes a virgin component from one the walk never reached and the first-allocation transition is defined on exactly that difference. It runs twice — once before the write (beside the orphan scan) and once inside the verify pass AFTER `SettleLandscapeLayers`, because the BASE allocation arrays are rewritten by the merge the settle drains, not by `SetAlphaData`. `DiffLandscapeComponentAllocationCoverage` then diffs them against the MATERIAL-declared layer set (`GetPaintableTargetLayerNames`), not `ULandscapeInfo::Layers`: a declared layer with no LayerInfo asset yet still compiles a `LandscapeLayerSample` node, and that node is what goes to `Constant(0.f)`. **Keyed by NAME, not by pointer, and that divergence from `ScanLandscapeOrphanedWeightAllocations` is deliberate and sourced:** the orphan scan mirrors `ReallocateLayersWeightmaps`, which compares `Alloc.LayerInfo` by pointer; this one mirrors the shader's static parameter set, which `GetCombinationMaterial` builds as `FStaticTerrainLayerWeightParameter(Allocation.GetLayerName(), ...)` (re-read at `LandscapeEdit.cpp:638-648` on the 5.8 source on this host). `FWeightmapLayerAllocationInfo::GetLayerName()` itself is NOT callable from this module — the struct carries no `LANDSCAPE_API` and the member is defined out of line at `LandscapeComponent.cpp:20` — so the name is read as `Alloc.LayerInfo->LayerName` under the deprecation pragma, the shape this file already uses at the orphan scan. **Published fields.** Always, because they are properties of the landscape and the call rather than of the write: `componentSizeQuads`, `componentSizeUU {x,y}` (= `Landscape->ComponentSizeQuads` × `GetActorScale3D()`, the same derivation `landscape.create` uses) and `requestedFootprintUU {sizeX,sizeY}` — the REQUEST, named separately per the house convention. Under `verify`: `componentsGainingAllocation`, `componentsGainingFirstAllocation`, `layersNowSampledZeroOnTouchedComponents[]` (`layerName` + `componentCount`, omitted when empty) and `allocationFootprintUU` (`minX`/`minY`/`maxX`/`maxY`/`sizeX`/`sizeY`, accumulated from `Component->Bounds.GetBox()`, OMITTED rather than zeroed when nothing gained — an all-zero box would read as a measured footprint at the world origin). **Deliberately gated on `verify` only, NOT on `bCensus`:** the walk is a pointer comparison, not a `GetWeightDataFast` read, so the 4 Mtexel census cap does not apply and a landscape too large for `layersAffected[]` still gets its blast radius. Without `verify` there is no settle, so the "after" read would measure the pre-merge state — the fields are then omitted, not zeroed, and the `verify` param text says so. **Ask 2 (the warning).** When `layersNowSampledZeroOnTouchedComponents[]` is non-empty, one `warnings[]` line names the counts, the sibling layers going to constant 0, the MEASURED footprint against the REQUESTED footprint with `componentSizeUU` beside them, and the ratio when measured exceeds requested — the "warn when measured and requested disagree" rule. It states plainly that nothing was erased and that `otherLayerTexelsLost: 0` is TRUTHFUL over this, then gives the order of work. Gated on the layer list rather than on `componentsGainingAllocation > 0`, because a component that already carried every declared layer gains nothing and loses nothing. **Ask 3 (the disclosure that read as a guarantee).** The registered summary's "the cost to the other layers is a reported number" is now "the WEIGHT cost", followed by a `THE BLAST RADIUS IS THE COMPONENT, NOT THE REGION` clause; the `verify` param text gains `AND NOT EVERY KIND OF DAMAGE: otherLayerTexelsLost is a DIFFERENCE OF TWO TEXEL COUNTS and cannot express a loss where no texel changed`; `Docs/wiki-src/landscape.md` gains the matching paragraph plus a field table, both placed after the existing orphan-gap paragraph so the two limits read as separate rather than as one closed by the other. **Ask 4 (order of work).** `paint the base layer over the FULL extent first, then accents over their regions` now sits in the registered summary, the new warning and the wiki page, stated as a property of the engine's per-component permutation rather than of any one map. **No refusal added**, per the ticket's own `Not asked for` — report and warn only. **Regression test** `PinWright.landscape.create_procedural_terrain.FirstPaintReportsComponentWideAllocationBlastRadius` in `Tests/Environment/TestLandscapePaintLayerHonesty.cpp`: 2x1-component landscape, material declaring two layers, NO prior paint so every component starts with an empty allocation array, then one paint of a 3x3-texel region inside component 0. It asserts BOTH halves in one place, which is the only shape in which this defect is expressible — `otherLayerTexelsLost == 0` (the census is clean and RIGHT to be), and simultaneously `componentsGainingFirstAllocation >= 1`, the unpainted sibling present in `layersNowSampledZeroOnTouchedComponents[]`, `allocationFootprintUU.sizeX` strictly greater than `requestedFootprintUU.sizeX` AND at least `0.9 ×` `componentSizeUU.x`, and a `warnings[]` line naming the sibling and the component-wide reach. Every one of those fails today on a missing field while `success` still reads `true`. The `otherLayerTexelsLost == 0` assertion is kept deliberately so a future change cannot "fix" the blast radius by breaking the census into reporting a phantom loss. `otherLayerTexelsLostInRegion` is NOT asserted, because `B-paint-auto-created-layer-never-weight-blended`'s fix makes in-region renormalisation legitimate. **Composition with that ticket, done second in the same session:** the two are independent by construction. Allocation is decided by `ReallocateLayersWeightmaps` from the requested target-layer set and does not consult a blend method, so making auto-created layers `FinalWeightBlending` cannot move any counter here; conversely this walk reads only allocation arrays and never a blend method. They share the response object and the registered summary, where the blend-method clause and the blast-radius clause are separate sentences. **NOT DONE, deliberately:** NOT COMPILED and NOT RUN — the orchestrator builds and runs. No live editor call and no re-measurement of the 6300 uu figure; the mechanism citations were re-read on this host's 5.8 source, the frame measurements were not repeated. World Partition is covered structurally only: the walk inherits `ForEachLandscapeProxy`'s loaded-proxies-only reach, which the existing `orphanScanLoadedProxiesOnly` flag and partitioned-world warning already disclose, but no partitioned landscape was exercised. The `GrassData` cached-grass-map question `#1` left open (why a zero-allocation component renders grass at all) was NOT closed and nothing here depends on it. `check_test_ids.py` and `check_test_skips.py` both re-run CLEAN.
