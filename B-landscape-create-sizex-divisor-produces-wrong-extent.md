---
id: B-landscape-create-sizex-divisor-produces-wrong-extent
title: "landscape.create's documented sizeX/sizeY conversion divides by a hardcoded 1000 uu per component, but a default component spans 8064 uu — a requested extent is silently built 8.064x too LARGE, and no scale/quadSize parameter exists to make the divisor true"
status: OPEN
severity: High
category: bug
tags: [landscape, create, sizeX, sizeY, hardcoded-divisor, draw-scale, silent-wrong-data, extent, missing-parameter]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# `sizeX` / `sizeY` convert through a divisor that no part of the spawned landscape agrees with

`landscape.create` (`LandscapeHandler.cpp:387`) accepts `sizeX` / `sizeY` and
documents them, verbatim at `:397` and `:398`:

> `sizeX` (`number`, optional): Landscape extent along X in world units, converted
> to componentsX as floor(sizeX / 1000) with a floor of 1. Ignored when componentsX
> is supplied.

The divisor is a literal, at `:440-446`:

```cpp
  double SizeXUnits = 0.0, SizeYUnits = 0.0;
  if (Payload->TryGetNumberField(TEXT("sizeX"), SizeXUnits) && SizeXUnits > 0 && !bHasCX) {
    ComponentsX = FMath::Max(1, static_cast<int32>(FMath::Floor(SizeXUnits / 1000.0)));
  }
  if (Payload->TryGetNumberField(TEXT("sizeY"), SizeYUnits) && SizeYUnits > 0 && !bHasCY) {
    ComponentsY = FMath::Max(1, static_cast<int32>(FMath::Floor(SizeYUnits / 1000.0)));
  }
```

`1000` asserts that one landscape component spans 1000 world units. Nothing in the
verb makes that true. A component's world span is
`ComponentSizeQuads * DrawScale.X`, and both factors are fixed by this handler:

- **`ComponentSizeQuads` is 63 at the defaults.** `quadsPerComponent` defaults to
  63 (`:399`, code `:448`), `sectionsPerComponent` defaults to 1 (`:400`, code
  `:453`) which maps to `NumSubsections = 1`, and `:498` computes
  `ComponentSizeQuadsField = NumSubsectionsField * SubsectionSizeQuadsField` = `1 * 63` = **63**.
- **`DrawScale.X` is 128.** This handler **never sets the actor scale** — there is
  no `SetActorScale3D` or `SetRelativeScale3D` anywhere in `LandscapeHandler.cpp`
  (the only scale references are the two *reads* at `:1031` and `:1955`). So the
  spawned `ALandscape` keeps the engine CDO value, set in the `ALandscapeProxy`
  constructor at `Engine/Source/Runtime/Landscape/Private/Landscape.cpp:1762`:
  `RootComponent->SetRelativeScale3D(FVector(128.0f, 128.0f, 256.0f));`
  `ALandscapeProxy::Import` — the call this verb builds the grid with (`:659` onward) —
  does not touch scale either; the engine's only scale write on the landscape path
  is in `PostEditChangeProperty` (`LandscapeEdit.cpp:6014`), which this verb never reaches.

**63 x 128 = 8064 uu per component. The divisor is off by a factor of 8.064,
and it is off in the direction that makes the landscape too big:**

| `sizeX` requested | `componentsX` = floor(sizeX/1000) | actual world X extent | ratio |
|---|---|---|---|
| 1 000 | 1 | 8 064 | 8.06x |
| 10 000 | 10 | 80 640 | 8.06x |
| 100 000 | 100 | 806 400 | 8.06x |

**This is independently confirmed on the wire, not only by arithmetic.**
`B-landscape-sculpt-stale-bounds` `#1-initial-repro` created a 2x2 / 63-quad /
1-section landscape through this same verb and read back
`actor.get_bounding_box` → `{"origin":[8064,8064,0],"extent":[8064,8064,0]}` —
a half-extent of 8064 for a 2-component axis, i.e. 16128 uu across, i.e.
**8064 uu per component**, exactly as derived here. That ticket cites the figure
for an unrelated purpose (its Z extent), which is why the X/Y half of it was
never read as a `sizeX` defect.

## Why this is silent

The success response (`:755-766`) echoes `landscapePath`, `actorLabel`,
`componentsX`, `componentsY`, `quadsPerComponent`, `subsectionSizeQuads`,
`numSubsections`, `componentSizeQuads`. **It never echoes a world extent, a
world size, or the draw scale** — the three things that would expose the
discrepancy. A caller who asked for `sizeX: 10000` and reads back
`componentsX: 10` has been told a true fact about a landscape 80,640 uu wide
and has no way to notice from the response that it is not the 10,000 uu one
they asked for. Everything downstream (actor placement, camera framing,
`region` coordinates in heightmap pixels) is then computed against the wrong
world footprint.

There is a second-order cost worth naming because it scales: the heightmap
array is allocated as `(ComponentsX * ComponentSizeQuads + 1)^2` uint16 at
`:642`. A caller asking for a 1 km landscape (`sizeX: 100000`) gets 100
components, i.e. a 6301 x 6301 vertex grid — about 79 MB of `uint16` and a
correspondingly large component build. Asking for 10 km would be ~7.9 GB.
That consequence is derived from `:642`, not observed; it is not the reason
this ticket is filed, but it is why the error does not stay cosmetic.

## Second defect on the same verb: no `scale` / `quadSize` parameter

The caller cannot correct the divisor themselves, because the term that makes
it wrong is not settable. The complete parameter list (`:389-402`, mirrored in
`Saved/PinWright/wiki/landscape.create.md`) is:

`name` (alias `landscapeName`), `location`, `x`, `y`, `z`, `componentsX`,
`componentsY`, `componentCount`, `sizeX`, `sizeY`,
`quadsPerComponent` (alias `quadsPerSection`), `sectionsPerComponent`, `materialPath`.

There is **no `scale`, `drawScale`, `quadSize`, `resolution` or units parameter**
of any kind. So a caller who wants a landscape whose components really are
1000 uu wide — which is what the docs describe — would need `DrawScale.X` =
1000/63 = 15.873, and there is no way to ask for it at create time. The two
knobs that *are* exposed (`quadsPerComponent`, `sectionsPerComponent`) move
`ComponentSizeQuads` only across {7, 15, 31, 63, 127, 255} x {1, 2}, so the
smallest reachable component is 7 x 128 = 896 uu and the largest is
255 x 2 x 128 = 65,280 uu. The divisor of 1000 is correct for **no** legal
combination.

## Workaround

Do not pass `sizeX` / `sizeY`. Compute the component count yourself and pass
`componentsX` / `componentsY`, which are honoured verbatim:

    componentsX = round(desiredExtentUU / (quadsPerComponent * numSubsections * 128))

At the defaults that is `round(desiredExtentUU / 8064)`. The granularity is
8064 uu, so an arbitrary extent is not reachable exactly — that is the
limitation the missing `scale` parameter would lift.

Setting the scale after the fact via `actor.set_transform` is **not** an
established workaround and should not be recorded as one: `SetActorScale3D`
(`ActorTransformHandler.cpp:62`) does not run `ALandscapeProxy::PostEditChangeProperty`,
which is where the engine mirrors a scale change into `ULandscapeInfo::DrawScale`
(`LandscapeEdit.cpp:6014-6019`), so the cached draw scale would go stale. Whether
that leaves a usable landscape is untested here and the ticket does not claim it does.

## What it should do

1. Derive the divisor instead of hardcoding it: `ComponentsX = round(sizeX / (ComponentSizeQuads * DrawScale.X))`,
   computed after `ComponentSizeQuadsField` is known (`:498`) and against the scale
   the actor will actually carry. `round` rather than `floor`, so a request lands on
   the nearest achievable extent rather than always short by up to one component.
2. Echo the achieved world extent (and the draw scale) in the success response at
   `:755-766`, so the residual quantisation is visible rather than silent. A caller
   asking in world units must be answered in world units.
3. Add the missing `scale` / `quadSize` parameter (section above), so a caller who
   needs an exact extent can set the term that decides it. Note this must go through
   the landscape's own scale-change path, not a bare `SetActorScale3D`, for the
   `ULandscapeInfo::DrawScale` reason above.
4. Until (1) ships, the parameter docs at `:397-398` are wrong and should say so
   rather than describing a conversion that produces an 8x error.

severity rationale: impact=**silent wrong data on a normal path** — a documented,
accepted parameter produces a landscape 8.064x the requested extent, the response
contains nothing that could reveal it, and every downstream world-space computation
is built on the wrong footprint (the rubric's High band: "the caller trusts a result
that is a lie and builds on it"). Reach: `landscape.create` is not an
almost-every-session method, so I **decline the reach bump-up** to Critical — and
Critical is in any case reserved for a crash or for corrupting/losing asset data,
which this is not (the landscape it builds is structurally valid, just the wrong
size). I also **decline the rare-edge-path bump-down** to Medium: `sizeX`/`sizeY` is
not an edge path but the documented front door for anyone who thinks in world units,
and the alternative (`componentsX`) requires knowing the 8064 figure, which is
documented nowhere. -> **High**.

## Same shape as

- `B-landscape-create-hollow-no-components` (IN-REVIEW, Critical) — same verb,
  **different axis**: whether the component grid gets built at all. That ticket is
  about `Import` never being called on 5.5+/5.7 so the actor spawns with zero
  `ULandscapeComponent`s; this one is about how many components get built and how
  wide each one is. Its fix (routing through `ALandscapeProxy::Import` on every
  engine version) does not touch the `sizeX` conversion or the draw scale, and
  the defect here reproduces on a landscape whose grid builds correctly.
- `B-landscape-create-inconsistent-subsection-geometry` (IN-REVIEW, High) — same
  verb, **different axis**: internal consistency of the three geometry fields
  (`ComponentSizeQuads == SubsectionSizeQuads * NumSubsections`) for a
  non-divisible `quadsPerComponent`/`sectionsPerComponent` pair. Its fix at `:490-498`
  is what makes `ComponentSizeQuads` reliably 63 at the defaults — i.e. it is
  *upstream* of this ticket's arithmetic and this ticket depends on it being
  correct. It says nothing about world units: every field it validates is a quad
  count, and the missing factor here is the draw scale, which it never reads.

## History
- `#1-divisor-vs-draw-scale` `OPEN` reporter — Filed from source, arithmetic re-derived at HEAD in this checkout; not RPC-replayed in this session. `landscape.create` (`LandscapeHandler.cpp:387`) documents `sizeX`/`sizeY` as `floor(size / 1000)` (`:397-398`) and implements it at `:440-446`, asserting 1000 uu per component. A default component actually spans 8064 uu: `ComponentSizeQuads` = 63 (`quadsPerComponent` default 63 at `:448`, `sectionsPerComponent` default 1 at `:453`, product at `:498`) times `DrawScale.X` = 128, which the handler never sets — grep confirms no `SetActorScale3D`/`SetRelativeScale3D` anywhere in the file, so the actor keeps the `ALandscapeProxy` CDO value from engine `Landscape.cpp:1762` (`FVector(128, 128, 256)`). So a requested extent is built **8.064x too LARGE**, not too small. Independently corroborated on the wire by `B-landscape-sculpt-stale-bounds` `#1`, whose 2x2 landscape from this same verb reads back `actor.get_bounding_box extent [8064,8064,0]` — 8064 uu per component, exactly the derived figure, cited there for its Z value so the X/Y implication went unread. Silent because the response (`:755-766`) echoes only component and quad counts, never a world extent or the draw scale. Second section: no `scale`/`drawScale`/`quadSize` parameter exists (full list `:389-402`), so the caller cannot make the divisor true — the reachable component sizes are 896..65280 uu and 1000 is correct for none of them. Dedup: board searches for `DrawScale` (4 files, all foliage/procedural instance scale — unrelated), `quadSize` (2 files, `B-remesh-size-param-ignored` and `F-retopo-secondary-params-ignored`, both geometry), `"world scale"` (0 files), `128,128,256` (0 files), `8064` (2 files: `B-landscape-sculpt-stale-bounds`, which reports the number without questioning it, and `B-niagara-reset-module-input-corrupts-stack`, coincidental). No ticket covers the `sizeX` conversion. Cross-linked to the two same-verb siblings above with the axis distinction stated for each.
