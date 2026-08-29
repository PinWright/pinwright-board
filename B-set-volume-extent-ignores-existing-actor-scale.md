---
id: B-set-volume-extent-ignores-existing-actor-scale
title: "volume.set_volume_extent's brush branch builds correct geometry but never normalizes the actor's existing scale, so the volume ends up extent x scale in every axis while the response echoes the requested extent — which breaks the one repair B-spawned-volumes-have-no-brush-geometry prescribes, because the verb that creates those volumes leaves a scale behind"
status: IN-REVIEW
severity: High
category: bug
tags: [volume, set_volume_extent, brush, actor-scale, silent-wrong-data, procedural-foliage, pcg, broken-workaround, no-readback]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The brush is built at the right size and then multiplied by a scale nobody reset

`volume.set_volume_extent` requested `extent {8000, 12000, 3000}` on an
`AProceduralFoliageVolume`. Measured after the call: actor scale `(80, 120, 30)`, bounds extent
`(640000, 1440000, 90000)`. The response reported `newExtent {8000, 12000, 3000}` and success.

The per-axis error is **80x, 120x and 30x** — not a uniform 100x. That asymmetry is the whole
diagnosis, and it is worth stating before anything else because the obvious reading of these
numbers is wrong.

## It is NOT the verb writing a second scale

`8000 / 100 = 80`, `12000 / 100 = 120`, `3000 / 100 = 30`, so the observed scale is numerically
identical to what `VolumeHandler.cpp:1272` would write — and that line is real:

    VolumeActor->SetActorScale3D(FVector(NewExtent.X / 100.0f, NewExtent.Y / 100.0f, NewExtent.Z / 100.0f));

**It did not run.** The two branches are mutually exclusive
(`Handlers/Volume/VolumeHandler.cpp:1265-1273`):

```cpp
1265:    ABrush* BrushVolume = Cast<ABrush>(VolumeActor);
1266:    if (BrushVolume)
1268:        CreateBoxBrushForVolume(BrushVolume, NewExtent);
1270:    else
1272:        VolumeActor->SetActorScale3D(FVector(NewExtent.X / 100.0f, ...));
```

`AProceduralFoliageVolume` -> `AVolume` -> `ABrush`, so `Cast<ABrush>` succeeds, `:1268` runs and
`:1272` is unreachable for this actor. Two independent confirmations that the brush branch is the
one that ran:

- **Arithmetic.** If `:1272` had run, the actor's bounds would be its *base* half-extent times 80.
  A non-brush `ATriggerBase` has a ~40-unit default half-extent (measured and recorded in
  `E-volume-set-extent-units-class-dependent-docs` `#2`), giving `3200`, not `640000`. The only
  base half-extent that yields `640000` at scale 80 is `8000` — which is exactly the brush
  `CreateBoxBrushForVolume` builds. A brush that exists means `Cast<ABrush>` succeeded, which
  means `:1272` was skipped. The two hypotheses are not both consistent with the data; only one is.
- **Source.** `grep -n "SetActorScale3D" VolumeHandler.cpp` returns exactly two hits, `:1272` and
  `:1471` (`volume.set_volume_bounds`), both inside the non-brush `else`. Neither brush helper
  touches scale: `VolumeHelpers::BuildBoxBrushGeometry` (`:90-127`) writes `PolyFlags`, `Brush`,
  `Brush->Polys`, `BrushComponent->Brush`, `BrushBuilder`, then `CubeBuilder->Build` and
  `FBSPOps::csgPrepMovingBrush` — and nothing else. `SpawnVolumeActor`'s brush overload
  (`:265-302`) does not set scale either.

## What actually happened: the scale was already there, written by the verb that created the volume

`foliage.create_procedural` spawns the volume and immediately writes
(`Handlers/Environment/FoliageHandler.cpp:1497-1499`):

```cpp
1497:  // AProceduralFoliageVolume uses ABrush with default extent of 100 units (half-size)
1498:  // Scale = desired_size / (default_brush_extent * 2) = desired_size / 200
1499:  Volume->SetActorScale3D(Size / 200.0f);
```

A scale of `(80, 120, 30)` is `Size / 200` for `Size = {16000, 24000, 6000}` — and `{8000, 12000,
3000}` is exactly half of that, i.e. the caller passed `set_volume_extent` the half-extent of the
same box it had asked `foliage.create_procedural` for. That is the natural thing to do, and it is
what makes the multiplier look like a clean `/100`: with `extent = size/2`, the leftover scale
`size/200` **equals** `extent/100` per axis. The coincidence is what makes this defect read as a
units bug in the verb rather than as an uncleared actor state.

Composed, the final half-extent is `extent * size/200`, i.e. `size^2 / 400` — quadratic in the
requested size, and per-axis. Check: `16000^2/400 = 640000`, `24000^2/400 = 1440000`,
`6000^2/400 = 90000`. All three match the measurement exactly.

So the defect in `volume.set_volume_extent` is a **missing write**, not a duplicated one: the brush
branch establishes geometry and never asserts the actor transform that geometry is rendered
through. For any volume whose scale is not `(1,1,1)`, the verb cannot deliver the extent it was
asked for, and nothing in its response says so.

## The response cannot reveal it

`newExtent` (`VolumeHandler.cpp:1279-1283`) re-emits `NewExtent`, the request, unconditionally —
it is not read back from the actor. `AddActorVerification` (`:1277`, defined
`Private/Utils/AssetUtils.cpp:1557-1585`) publishes `actorPath`, `mapPath`, `actorName`,
`actorLabel`, `actorObjectName`, `actorGuid`, `existsAfter: true`, `actorClass` — **no bounds and
no scale**. A caller has to make a separate `actor.get_bounding_box` call to discover that the
volume is six orders of magnitude off by volume.

The blind `newExtent` echo itself is already owned by
`B-blocking-volume-no-brush-geometry` `#7-additional-trigger-sphere-extent-echo-mismatch`
(IN-REVIEW), which asks for `newExtent` to reflect the post-apply `GetActorBounds()` half-extent.
Not re-filed here. It is named because that fix, if it lands, is also the detector for this one:
an echo derived from real bounds would have reported `640000` and this ticket would have been a
one-line surprise instead of a live investigation.

## Why this invalidates a recommended workaround, and what the correction is

`B-spawned-volumes-have-no-brush-geometry` (OPEN, High, filed this session) documents that
`foliage.create_procedural` and `actor.spawn` produce `ABrush` volumes with no brush model, and
prescribes under **## Workaround**:

> `volume.set_volume_extent` repairs both after the fact: it takes the `Cast<ABrush>` branch and
> calls `CreateBoxBrushForVolume` (`VolumeHandler.cpp:1268`), which builds the model the spawn
> skipped. So the sequence is spawn -> `volume.set_volume_extent` -> only then use the volume.

That sequence is **not sufficient**, and the reason is inside that same ticket: it documents the
`SetActorScale3D(Size / 200.0f)` write at `FoliageHandler.cpp:1499` as a scale applied to geometry
that does not exist. It is right that the write is inert *at spawn time*. What it does not follow
through on is that the write is **persistent**: once `set_volume_extent` supplies the geometry, the
scale stops being inert and starts multiplying. Building the brush is what activates it.

The correction is to reset the actor scale to `(1,1,1)` in the same sequence — either order works,
because `BuildBoxBrushGeometry` derives nothing from the transform:

    foliage.create_procedural -> actor.set_transform (scale 1,1,1) -> volume.set_volume_extent

**This correction is UNTESTED.** No RPC call was made to verify it. What supports it is arithmetic
and source: the half-extent the brush branch produces is `NewExtent` (`CreateBoxBrushForVolume`
`:131-134` doubles it into a box size, `UCubeBuilder` `:115-119` builds that box centred on the
actor), and a unit scale leaves it alone. What is *not* established is whether
`AProceduralFoliageVolume`'s `UProceduralFoliageComponent` samples the brush bounds or the
component bounds, and whether either needs a further refresh after the transform write. A fixer
should call it before repeating it.

## Fix

In the brush branch of both `volume.set_volume_extent` (`VolumeHandler.cpp:1266-1269`) and
`volume.set_volume_bounds` (`:1465-1469`), set the actor scale to `FVector::OneVector` alongside
the brush build, so the extent the caller asked for is the extent the actor has. Do it in the
handler rather than inside `BuildBoxBrushGeometry`: the helper is also called from the spawn path
(`:298`) where the actor is freshly spawned at unit scale and the write would be dead code, and
from the sphere/capsule wrappers (`:136-144`).

If normalizing the scale is considered too aggressive (a caller may have scaled a brush volume
deliberately), then the verb must at minimum **report** it: an `appliedScale` field, or a
`warnings[]` line naming the residual scale and the resulting bounds. What must not survive is a
success payload that echoes an extent the volume does not have. The two are not equal options —
silently deferring to a stale transform is how this defect stayed invisible through a spawn, a
repair and a readback.

Deleting `FoliageHandler.cpp:1497-1499` is the other half and belongs to
`B-spawned-volumes-have-no-brush-geometry`, which already asks for exactly that. Fixing only that
side would leave every volume already on disk carrying the scale.

## Same shape as

`B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) carries the fullest statement of the
class: the call succeeds, every number it reports is correct, and the output is wrong because the
deciding number was never reported. Here the deciding number is the actor's own scale — present in
the level, absent from every input and every output of the verb that depends on it.

Nearest members:

- `B-spawned-volumes-have-no-brush-geometry` (OPEN, High) — **read this ticket with that one or
  neither makes sense.** Its workaround is the repro for this defect, and this defect is why its
  workaround does not work. Bidirectional: that ticket needs a note that the repair requires a
  scale reset; this one exists because the repair does not include one.
- `B-blocking-volume-no-brush-geometry` (IN-REVIEW, Critical) — its `#3-fix-brush-model-init`
  introduced `BuildBoxBrushGeometry`, and that fix is correct. This is not a defect *in* it: the
  helper does exactly what it says. It is a defect in the interaction between that new geometry
  and the pre-existing `SetActorScale3D` write two files away, which had no observable effect
  until the geometry existed. A fix that makes an inert bug live is still the fix's problem to
  notice. Its `#7` is the response-side echo, named above.
- `E-volume-set-extent-units-class-dependent-docs` (OPEN, Low) — very likely the historical reason
  `:1272` exists at all: `extent` means an absolute half-extent on the brush branch and `scale x
  100` on `ATriggerBase`. That ticket asks for the split to be documented. This one is a
  consequence of the split being real in code: because the brush branch is the one that does *not*
  own the scale, nobody owns clearing it.

## What was NOT done

- No source was modified.
- The `(1,1,1)` correction was **not** executed against the editor. See above for what backs it
  and what does not.
- Only `AProceduralFoliageVolume` was measured. `volume.set_volume_bounds` (`:1465-1469`) has the
  identical brush branch and the identical omission by source read, and was not called.
- The defect requires a pre-existing non-unit scale. A volume created by `volume.create_*` spawns
  at unit scale (`SpawnVolumeActor` `:265-302` writes no scale), so `set_volume_extent` is correct
  on those — which is why this survived: the verb is right on the path it was tested on and wrong
  on the path it was prescribed for.

severity rationale: impact=High — the README's High band verbatim, "silent wrong / stale / hardcoded data on a normal path (the caller trusts a result that is a lie and builds on it)". The call returns success, echoes `newExtent` equal to the request (`:1279-1283`), and `AddActorVerification` (`AssetUtils.cpp:1557-1585`) publishes neither bounds nor scale, so there is no field in the response a caller could check. For a procedural-foliage or PCG host the volume is the sampling region, so a volume 80-120x oversized in XY samples the whole level: the downstream symptom is a scatter or a generate over the wrong area, or a simulation that does not return, and neither points back at this verb. NOT Critical: the Critical band names an editor crash or a write that corrupts or loses asset data, and neither is present — the volume is written exactly as constructed, its transform and brush are both internally valid, nothing pre-existing is damaged, and deleting or rescaling the actor is a complete recovery. Wrong is not corrupt. I considered and reject the argument that an enormous, immediately-visible error rates below silent wrong data: it is visible in a viewport to a human, and this board's callers are agents that read the response — where the error is not merely unreported but actively contradicted by `newExtent`. Detection needs a second verb (`actor.get_bounding_box`) that nothing in the response suggests calling. NOT Medium: there is no documented workaround to be "doable via", because the documented workaround is the repro. x reach: BOTH modifiers declined. Not a bump up — `volume.set_volume_extent` is one of three configuration verbs in one namespace and is not in almost every session. Not a bump down — it is emphatically not a rare edge path: it is the single repair prescribed on this board for the zero-extent volumes two other verbs produce, so the path it breaks is the recovery path, which is reached precisely when something has already gone wrong. High stands unmodified.

## History
- `#1-brush-branch-leaves-stale-scale` `OPEN` reporter — Measured live against a running editor. `volume.set_volume_extent {extent:{8000,12000,3000}}` on an `AProceduralFoliageVolume` returned success with `newExtent {8000,12000,3000}`; the actor read back scale `(80,120,30)` and bounds extent `(640000,1440000,90000)` — per-axis errors of **80x, 120x, 30x**, NOT the uniform 100x it superficially resembles. All line numbers re-derived this session against HEAD. Mechanism, and it is not the obvious one: `VolumeHandler.cpp:1265-1273` is an exclusive if/else, `AProceduralFoliageVolume` is an `ABrush`, so `CreateBoxBrushForVolume` (`:1268`) ran and `SetActorScale3D(NewExtent/100)` (`:1272`) did NOT. Confirmed two ways — (a) arithmetic: `640000 = 8000 x 80` requires a base half-extent of 8000, which only the brush provides; a non-brush `ATriggerBase`'s ~40-unit default would have given 3200; (b) source: the only two `SetActorScale3D` calls in the file (`:1272`, `:1471`) are both inside non-brush `else` branches, and `BuildBoxBrushGeometry` (`:90-127`) plus `SpawnVolumeActor` (`:265-302`) write no scale at all. The scale was **pre-existing**, written by `foliage.create_procedural` at `FoliageHandler.cpp:1499` (`SetActorScale3D(Size / 200.0f)`, comment `:1497-1498`): `Size {16000,24000,6000}` gives exactly `(80,120,30)`, and the requested extent is exactly `Size/2`, which is why the leftover `Size/200` coincides per-axis with `extent/100` and disguises the defect as a units bug. Composed final half-extent is `Size^2/400` — quadratic and per-axis; `16000^2/400=640000`, `24000^2/400=1440000`, `6000^2/400=90000`, all three match. So the defect is a MISSING write: the brush branch establishes geometry and never normalizes the transform that geometry is rendered through. The response cannot reveal it — `newExtent` (`:1279-1283`) re-emits the request and `AddActorVerification` (`:1277`, `AssetUtils.cpp:1557-1585`) publishes no bounds and no scale. **This invalidates the workaround `B-spawned-volumes-have-no-brush-geometry` prescribes** (spawn -> `set_volume_extent` -> use): that ticket correctly notes the `:1499` scale write is inert against a null brush, but the write is persistent, so building the brush is what activates it. Correction is to reset actor scale to `(1,1,1)` in the same sequence (either order — the builder derives nothing from the transform); **that correction is UNTESTED**, no RPC verified it, and what is unestablished is whether `UProceduralFoliageComponent` samples brush or component bounds and whether either needs a refresh after the transform write. NOT DONE: no source modified; `volume.set_volume_bounds` (`:1465-1469`, identical branch, identical omission) implicated by source read only and not called; only `AProceduralFoliageVolume` measured. Dedup: `grep -ril` for `set_volume_extent` and `SetActorScale3D` across the board returns nine tickets, none of which claims the brush branch leaves a stale scale — `B-blocking-volume-no-brush-geometry` `#7` is the response-side `newExtent` echo (cited, not re-filed), `E-volume-set-extent-units-class-dependent-docs` is the request-side brush-vs-non-brush units split (cited as the likely reason `:1272` exists), and `B-spawned-volumes-have-no-brush-geometry` documents the `:1499` write only in its inert state. No umbrella filed; the class statement stays in `B-foliage-paint-does-no-ground-projection` and is referenced. Filed under this id rather than the proposed `B-set-volume-extent-applies-scale-on-top-of-brush` because that name asserts the mechanism the source rules out, and ids are quoted verbatim by sibling tickets.
- `#2-normalize-scale-and-measure-back` `IN-REVIEW` developer — **Contract decided and stated: `extent` is a WORLD half-extent, not a local one.** Three things force it and they agree: `volume.set_volume_bounds` shares this exact branch and derives its extent from world min/max corners plus a world centre write, where nothing but world is coherent; the non-brush `else` already overwrites the actor scale outright (`extent/100`), so the verb has always claimed ownership of the transform and treated `extent` as an absolute size; and callers reach this verb with the same numbers they gave a create verb, which spawns at unit scale where local and world coincide. Implemented by normalizing the actor scale to `FVector::OneVector` in the brush branch of BOTH `volume.set_volume_extent` and `volume.set_volume_bounds` — the ticket's own prescription — rather than dividing the requested extent by the residual scale: division preserves a scale the verb does not otherwise honour, and blows up on a zero or mirrored axis where no world extent is achievable at all. **The brush does NOT need the `B-shape-extent-stale-physics` treatment, and the reason is ordering.** That ticket's `FBodyInstance::UpdateBodyScale(Scale3D, bForceUpdate=true)` was needed because a bare `BoxExtent` reflection store recreates no state; here `CreateBoxBrushForVolume` -> `BuildBoxBrushGeometry` ends in `FBSPOps::csgPrepMovingBrush` -> `UBrushComponent::BuildSimpleBrushCollision`, whose `#if WITH_EDITOR` tail is `BrushBodySetup->CreateFromModel(Brush, true)` followed by an unconditional `RecreatePhysicsState()` (`BrushComponent.cpp:730-758`) — a full destroy-and-rebuild of the body from the transform in force at build time, strictly stronger than an `UpdateBodyScale`. Writing the scale FIRST and building second therefore leaves no stale window; the reverse order would have needed the shape treatment (or would have relied on `UPrimitiveComponent::OnUpdateTransform`'s `BodyInstance.UpdateBodyScale(GetComponentTransform().GetScale3D())`, `PrimitiveComponent.cpp:1082`, which does fire here because the scale genuinely changes — but that is a second body build for no reason). Response now follows the `LightingHandler` measured-beats-requested convention: `newExtent` is `GetActorBounds()` read back AFTER the apply instead of an unconditional echo of the request, `requestedExtent` names what the call asked for, and `clearedScale` names the non-unit scale that was reset — **omitted, never zeroed**, when there was none, so a unit value cannot be confused with "the verb cleared a scale that happened to be unit". `set_volume_bounds` gets the same shape: `bounds` and `center` measured off the actor, `requestedBounds` beside them, `clearedScale` on the same omit rule. This subsumes what `B-blocking-volume-no-brush-geometry` `#7` asked for on `set_volume_extent`'s echo (that ticket is IN-REVIEW but the raw echo was still in HEAD when this was written); its separate non-brush units question — `extent` meaning scale x 100 against a ~40-unit base on `ATriggerBase` — is NOT addressed here and stays with `#7` / `E-volume-set-extent-units-class-dependent-docs`, though the measured `newExtent` now makes that mismatch visible in the response instead of contradicted by it. Files: `Handlers/Volume/VolumeHandler.cpp` (both handlers plus two file-local JSON emitters, `MakeVolumeVectorJson` / `MakeVolumeBoundsJson`; the verb descriptions now state the world contract and the rotated-AABB caveat), `Tests/World/TestVolumeHandlers.cpp` (+270). Regression tests are behavioural, not shape-only: `PinWright.volume.set_volume_extent.NormalizesExistingActorScale` and `PinWright.volume.set_volume_bounds.NormalizesExistingActorScale` spawn an `ABlockingVolume` (a real `ABrush`, unlike the `ATriggerBox` that `#8` corrected), plant a deliberately PER-AXIS scale `(2,3,4)` so a uniform defect and a per-axis one cannot be confused, call the verb, then assert the MEASURED `GetActorBounds()` world half-extent equals the request, that the actor reads unit scale afterwards, that `newExtent`/`bounds` equal the measurement rather than the request, and that `requestedExtent`/`requestedBounds`/`clearedScale` are present and correct. With the fix reverted every one of those fails — the bounds read `(2x,3x,4x)`, the scale is still `(2,3,4)`, and the three new fields are absent. NOT DONE: not compiled and not run (the wave builds and runs the suite afterwards), and no live RPC replay against `AProceduralFoliageVolume` — so the ticket's open question of whether `UProceduralFoliageComponent` samples brush or component bounds after the transform write is still unanswered, and the `RecreatePhysicsState()` reasoning above is a source read, not a measurement. Contradicting the ticket, minor and in its favour: the mid-ticket claim that either order works because `BuildBoxBrushGeometry` derives nothing from the transform is true of the GEOMETRY but not of the physics body, which is rebuilt from the transform at build time — the order is not free, and this fix pins it. Also note the helpers the ticket cites at `VolumeHandler.cpp:90-127` have since moved to `Handlers/Volume/VolumeBrushGeometry.h` (a concurrent `B-spawned-volumes-have-no-brush-geometry` change, re-exported into `VolumeHelpers` by `using`), so every line number in this ticket above is stale — re-locate by symbol.
