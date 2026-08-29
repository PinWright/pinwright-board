---
id: E-create-grass-type-cull-distance-and-output-path
title: "landscape.create_grass_type writes StartCullDistance == EndCullDistance so grass pops in with no fade band, takes no cull-distance parameters, hardcodes its output path to /Game/Landscape, and never reaches disk — at the default 10000 the carpet stops inside a 160 x 240 m zone"
status: OPEN
severity: Medium
category: ergonomic
tags: [landscape, create_grass_type, grass, vegetation, cull-distance, fade-band, hardcoded-path, save-no-disk-write, performance-knob, parameter-gap]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The one number that decides whether the grass reaches the far edge is not a parameter

`landscape.create_grass_type` (`Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp:1502`)
declares five parameters — `name`, `meshPath`, `density`, `minScale`, `maxScale` (`:1503-1509`) —
and none of them is a cull distance. Four separate consequences follow from that block and the two
lines around it.

## 1. `StartCullDistance == EndCullDistance`, so grass pops instead of fading

The variety-construction block (`:1587-1606`) writes `GrassMesh`, both density slots,
`ScaleX/Y/Z`, `RandomRotation` and `AlignToSurface`. It never touches `StartCullDistance` or
`EndCullDistance`, so both keep the constructor default:

```
FGrassVariety::FGrassVariety()
    ...
    , StartCullDistance(10000)
    , StartCullDistanceQuality(10000)
    , EndCullDistance(10000)
    , EndCullDistanceQuality(10000)
```

`C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeGrass.cpp:1486-1489`.

Equal start and end collapses the fade band to zero width: instances are at full size right up to
the cull radius and then gone. Every grass type this verb produces has that artefact, and no
parameter on the verb can prevent it.

## 2. The measured consequence: the carpet stops inside the authored zone

This is what makes it more than a parameter wish. At the default 10000 the grass carpet stops
**inside** a 160 x 240 m zone — bare ground in the far half of the area the type was created for.

The zone agent proved this was the cull distance and **not** a weightmap edge by moving only the
camera, holding the weightmap fixed:

| camera distance to the ground point | result |
|---|---|
| 3,905 uu | full carpet |
| 12,903 uu | bare ground |

Same weightmap, same landscape, one camera move. Retuning to `12000 / 20000` fixed it.

## 3. The cull distance is also the performance knob, which is the argument for exposing it

Retuning to `12000 / 20000` cost roughly **4x the instance count**. That is the expected shape: the
grass build radius is `EndCullDistance` and the built area grows with its square, so this single
number sets both the visible extent and the cost.

A caller who cannot pass it has neither control. They cannot make the grass reach the far edge of
their zone, and they cannot trade extent for instance count when it is too expensive. Raising the
handler's default instead would fix the first at the cost of the second, on every map, silently —
which is why the ask is a parameter and not a bigger default.

## 4. The output path is hardcoded

`:1556`:

```cpp
FString PackagePath = TEXT("/Game/Landscape");
```

Every grass type this verb creates lands in `/Game/Landscape/<name>`, with no parameter to change
it. The obvious remedy — create it, then `asset.move` it somewhere sensible — currently destroys
the asset's name: see `B-asset-move-to-missing-folder-silently-renames` (OPEN, High), filed the same
session, where `asset.move` into a non-existent folder renames rather than moves and reports
success. So the workaround for this gap runs straight into a live defect.

## 5. It does not persist to disk

`:1608` calls `McpSafeAssetSave(GrassType)`, which is mark-dirty only —
`Source/PinWright/Private/Utils/AssetUtils.cpp:362-377`:

```cpp
void McpSafeAssetSave(UObject* Asset)
{
    if (!Asset) return;
    // UE 5.7+ Fix: Do not immediately save newly created assets to disk. ...
    Asset->MarkPackageDirty();
    FAssetRegistryModule::AssetCreated(Asset);
}
```

No write. The header's own note (`Utils/AssetUtils.h:107`, `:114`, `:206`, `:220`) says the same,
and the function was deliberately made `void` because "the mark always succeeds and never persists
anything, so a bool return could only ever have been the constant `true` — which is exactly what
callers were publishing as `saved`". The response here does not claim `saved`, which is correct,
but it also does not say the asset is unwritten.

This half is the `*-save-no-disk-write` family, whose head is `B-audio-create-save-no-disk-write`;
the sibling files are `B-material-authoring-save-no-disk-write`,
`B-metasound-create-save-no-disk-write`, `B-niagara-save-no-disk-write`,
`B-pose-search-create-save-no-disk-write`, `B-sequencer-create-save-no-disk-write` and
`B-texture-save-no-disk-write`. Recorded here for the grass verb rather than filed separately,
because it is one line of one handler and belongs with the other four asks about the same block.

## Ask

1. `startCullDistance` / `endCullDistance` parameters, both optional, defaulting to the engine
   constructor's 10000 so nothing already built changes. Write both the per-platform and the
   per-quality slot, exactly as the handler already does for density at `:1600-1601` — and for the
   same reason recorded in the comment there: `GetDensity()` / `GetEndCullDistance()` read the
   quality slot on hosts with `GEngine->UseGrassVarityPerQualityLevels`, so writing one leaves the
   constructor's value in force on half the hosts.
2. Report both back off the stored variety. The handler already does exactly this for
   `end_cull_distance` at `:1617` — "read the two numbers the engine gates on back off the stored
   variety rather than echoing the request" — so `start_cull_distance` is one line beside it.
3. A `packagePath` parameter, defaulting to `/Game/Landscape`.
4. Persist, or say it did not. Either write the asset, or publish the measured
   `AddMarkDirtySaveReport` block the rest of the surface uses (`AssetUtils.cpp:397`), so a caller
   knows the `.uasset` is not on disk yet.

## Same shape as

- `B-create-grass-type-addzeroed-never-renders` (DONE, High) — the same handler, the same variety
  block, the same class of "a field the constructor sets and the handler does not". **This is a
  follow-on, not a reopening.** That ticket was about `AddZeroed` leaving `EndCullDistance` at 0,
  which made the variety render nothing at all; its fix (`AddDefaulted_GetRef` at `:1587`) is
  present in this tree and is exactly what puts the constructor's 10000 into both slots. This ticket
  starts from that fixed state and says 10000 is not always the right answer, that the equal
  start/end is an artefact, and that the caller cannot change either.
- `B-audio-create-save-no-disk-write` and the six sibling `*-save-no-disk-write` tickets — the
  persistence half, item 5.
- `B-asset-move-to-missing-folder-silently-renames` (OPEN, High) — the defect the hardcoded-path
  workaround runs into.

## Severity

**Medium**, by impact class: *soft blocker* — "doable, but only via a documented workaround, a
source dive, or many extra calls". Every one of the four gaps has a route around it: edit the
variety afterwards through `property.set`, move the asset afterwards, save it afterwards. All of
them are extra calls the caller has to know to make, and two of them require knowing that
`EndCullDistance` is what stopped the carpet.

**Reach modifier declined.** `landscape.create_grass_type` is a rare path — one call per grass type
— which by the rubric would bump this down to Low. Declined because Low is "pure friction: docs,
discoverability, naming, cosmetic", and the measured outcome here is not cosmetic: at the shipped
default the grass visibly stops inside the zone it was authored for, and a caller who does not know
the cull distance is the cause will look at the weightmap. A verb whose default produces a visibly
wrong result is not friction.

Not rated High: the asset is valid, the verb reports honestly, and nothing here is a false success.
The gaps are missing knobs and a missing write, not a lie.

## History
- `#1-cull-distance-not-a-parameter` `OPEN` reporter — Filed off live measurement. At the default
  `EndCullDistance` 10000 the grass carpet stops inside a 160 x 240 m zone; proved to be the cull
  distance and not a weightmap edge by moving only the camera against a fixed weightmap — bare
  ground at 12,903 uu, full carpet at 3,905 uu. Retuning to 12000/20000 fixed it at roughly 4x the
  instance count, the build radius being `EndCullDistance` and cost growing with its square, which
  is the argument for a parameter over a larger default. Source re-derived: the handler registers
  five parameters, none a cull distance (`LandscapeHandler.cpp:1502`, params `:1503-1509`); the
  variety block (`:1587-1606`) never assigns `StartCullDistance` or `EndCullDistance`, so both keep
  the constructor's 10000 (`LandscapeGrass.cpp:1486-1489`) and start == end leaves no fade band;
  the output path is hardcoded at `:1556`; and `McpSafeAssetSave` at `:1608` is mark-dirty only
  (`AssetUtils.cpp:362-377`). Cross-linked `B-create-grass-type-addzeroed-never-renders` as the
  predecessor whose fix produced this starting state, and `B-asset-move-to-missing-folder-silently-renames`
  as the defect the hardcoded-path workaround hits.
