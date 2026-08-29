---
id: B-lightmass-volume-no-brush-geometry
title: "lighting.create_lightmass_volume spawns ALightmassImportanceVolume with no brush model, so the importance region is a POINT — and because the engine's fallback is gated on the volume COUNT, calling this verb makes a lighting build worse than never calling it"
status: IN-REVIEW
severity: High
category: bug
tags: [volume, brush, lightmass, importance-volume, lighting, build-lighting, silent-false-success, zero-extent, volumetric-lightmap, spawn-path]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The fourth instance of the phantom-volume defect, on the one volume class where the phantom is worse than the absence

`B-spawned-volumes-have-no-brush-geometry` `#4-lifted-brush-builder-to-both-spawn-paths` fixed two
spawn paths and named this one as a deliberately-unfixed fourth instance wanting its own ticket.
This is that ticket. **It still stands at HEAD** — see [Why the sibling's fix does not reach
it](#why-the-siblings-actorspawn-arm-does-not-reach-this-verb).

**Name correction, carried forward from that entry:** the verb is registered as
**`lighting.create_lightmass_volume`**, not `lighting.create_lightmass_importance_volume`. That
second name belongs to a *different, working* verb in the `volume` namespace — see
[Workaround](#workaround). Anyone grepping for the id in that entry will not find a handler.

## Mechanism

`Handlers/Environment/LightingHandler.cpp`, the `REGISTER_RPC_HANDLER("lighting.create_lightmass_volume", ...)`
body:

```cpp
    AActor* Volume = SpawnActorInActiveWorld<AActor>(
        ALightmassImportanceVolume::StaticClass(), Location, FRotator::ZeroRotator);
    if (Volume)
    {
        PinWright::MarkLevelActorSpawned(Volume);
        Volume->SetActorScale3D(Size / 200.0f);
```

`ALightmassImportanceVolume : public AVolume` (Engine:
`Runtime/Engine/Classes/Lightmass/LightmassImportanceVolume.h`), so it is an `ABrush` that keeps
its shape in a `UModel`, not in its transform. This is a raw spawn: `Brush` is null, `Polys` is
null, `BrushComponent->Brush` is unwired. `grep -c "Brush" LightingHandler.cpp` is **0** across the
whole file — no `ABrush`, no `UModel`, no `UPolys`, no `VolumeBrushGeometry` include.

The bounds that follow are engine-determined and exact. `UBrushComponent::CalcBounds`
(Engine: `Runtime/Engine/Private/Components/BrushComponent.cpp`) tests `Brush && Brush->Polys &&
Brush->Polys->Element.Num()`, then `BrushBodySetup->AggGeom.GetElementCount() > 0`, and with
neither present returns the final `else`:

```cpp
        return FBoxSphereBounds(LocalToWorld.GetLocation(), FVector::ZeroVector, 0.f);
```

A **point** at the actor's location. Note `LocalToWorld` is applied to the *origin* only — the
extent is a hard `FVector::ZeroVector` that no actor scale can multiply, which is why the
`SetActorScale3D` line above is currently inert.

The response is `success: true` plus `AddActorVerification` (`Utils/AssetUtils.cpp`), which writes
`actorPath` / `actorName` / `actorGuid` / `actorClass` / `existsAfter: true` and **no bounds field
of any kind**. `existsAfter` is true for a phantom. Nothing in the payload can distinguish this
from a working volume.

## Why the sibling's `actor.spawn` arm does not reach this verb

The `#4` fix added `else if (AVolume *VolumeActor = Cast<AVolume>(Spawned))` →
`VolumeBrushGeometry::BuildBoxBrushGeometry(VolumeActor, FVector(DefaultBoxSize))` to
`Handlers/Actor/SpawnHandler.cpp`. That arm is inside the **`actor.spawn` handler body**, in the
post-spawn chain following that file's own file-local `SpawnActorInWorld` helper. It is not in a
shared helper and nothing outside that handler can reach it.

`lighting.create_lightmass_volume` does not call `actor.spawn`. It calls the template
`SpawnActorInActiveWorld<T>` declared and defined in `Utils/AssetUtils.h` — a different spawn
helper in a different file, whose body is a plain
`World->SpawnActor(ActorClass, &Location, &Rotation, SpawnParams)` under a PIE/editor branch.
`grep -n "Brush\|AVolume" Utils/AssetUtils.h` returns **nothing**. The gate the sibling installed
sits one layer above the layer this verb enters at, so the two never meet.

The same negative result applies to every other caller of `SpawnActorInActiveWorld` on an
`ABrush` subclass. Only `PostProcessVolumeUtils.h` is known to be one, and `#4` already cleared it
(it sets `bUnbound = true`, so bounds are irrelevant there).

## Consequence — specific to a Lightmass importance volume, and worse than the generic case

The generic phantom-volume consequence is "the volume does nothing." Here it is **"the volume
actively replaces something that worked."** The chain, all in the editor-side exporter:

**1. The engine's automatic fallback is gated on the volume COUNT, not on any extent.**
`FStaticLightingSystem::GatherScene`
(Engine: `Editor/UnrealEd/Private/StaticLightingSystem/StaticLightingSystem.cpp`) iterates
`TObjectIterator<ALightmassImportanceVolume>` and calls
`FLightmassExporter::AddImportanceVolume` for each, then:

```cpp
	if (LightmassExporter->GetImportanceVolumes().Num() == 0)
	{
		FBox ReasonableSceneBounds = AutomaticImportanceVolumeBounds;
		...
		LightmassExporter->AddImportanceVolumeBoundingBox(ReasonableSceneBounds);
	}
```

With **no** importance volume in the level, the engine synthesizes one from the scene bounds
(expanded by `AutomaticImportanceVolumeExpandBy=500`, `BaseLightmass.ini`) and logs a Warning
suggesting the user add a real one — the build is slower but **correct**. With the phantom present
the count is 1, the synthesis branch is skipped, and the point box is shipped as the entire
importance region. `FLightmassExporter::AddImportanceVolume` (`Lightmass.h`) stores
`InImportanceVolume->GetComponentsBoundingBox(true)` verbatim, so the degenerate bounds pass
through unexamined; `Lightmass.cpp` writes `Scene.NumImportanceVolumes = ImportanceVolumes.Num()`
and streams the boxes to Swarm with no validity check. **This is the load-bearing point of the
ticket: the verb does not fail to help, it removes help that was already there.**

**2. Indirect lighting outside the point is switched off.** `BaseLightmass.ini` ships
`bEmitPhotonsOutsideImportanceVolume=False`, documented in Lightmass's own
`Programs/UnrealLightmass/Public/SceneExport.h` as: *"If this is false, nothing outside the volume
will bounce lighting and will be lit with direct lighting only."* The companion
`OutsideImportanceVolumeDensityScale=.0005` scales outside-gather density to 1/2000. Both are
measured against a region of zero volume, so the whole level is "outside."

**3. The volumetric lightmap collapses to one brick anchored at the spawn point.**
`FLightmassExporter::SetVolumetricLightmapSettings` (`Lightmass.cpp`) unions the importance volumes
into `CombinedImportanceVolume`, takes `ImportanceExtent = GetExtent()` — `(0,0,0)` — and computes
`FullGridSize = TruncToInt(2 * ImportanceExtent / TargetDetailCellSize) + 1` = `(1,1,1)`, with
`OutSettings.VolumeMin = CombinedImportanceVolume.Min` = the spawn point. At stock settings
(`BrickSize=4`, `MaxRefinementLevels=3` → `DetailCellsPerTopLevelBrick = 64`;
`VolumetricLightmapDetailCellSize` default 200, `WorldSettings.h`), `TopLevelGridSize` rounds up to
`(1,1,1)` and `VolumeSize` = 64 x 200 = **12800 uu on each axis, with its MIN corner at the
phantom's location** — not centred on it, and not covering the level. Every movable/dynamic object
outside that one cube gets no baked indirect lighting.

So the cost of a build after this verb is: a full-length bake that produces flat, direct-only
static lighting over the level, a volumetric lightmap covering one arbitrary 12800 uu cube, no
warning in `LightingResults` (the "No importance volume found" message log and its Warning-level
sibling are both inside the skipped branch), and `MapBuildData` overwritten with that result. The
build reports success; a previously-correct bake in the same map is gone.

## The `Size / 200.0f` scale is the same matched pair the sibling found

`SetActorScale3D(Size / 200.0f)` is inert today — `CalcBounds` returns a literal
`FVector::ZeroVector` extent on the null-brush path, and scale cannot multiply zero. It is **not**
inert after a naive fix. A fixer who adds `BuildBoxBrushGeometry(Volume, Size)` and leaves this
line gets bounds of `Size/2 x (Size/200)` per axis — an overshoot factor of `Size/200` on **each
axis independently**. At the verb's own default `size` of `(1000,1000,1000)` that is 5x per axis
(125x by volume); at a level-sized `size` of `20000` it is 100x per axis, the same magnitude
`B-spawned-volumes-have-no-brush-geometry` `#3-correcting-my-own-2-the-scale-is-this-tickets-own-defect`
measured on the foliage verb. **Delete the line with the fix, exactly as `#4` did in
`FoliageHandler.cpp`.** Any migration of volumes already produced by this verb must also normalize
scale to `(1,1,1)`: every one of them is carrying `Size/200`.

An oversized importance volume is not the safe direction here either — it is what
`MinimumImportanceVolumeExtentWithoutWarning=10000.0` and the "tightly bounding" advice in the
engine's own message exist to prevent, and it inflates `FullGridSize` toward the
`MaxGridDimension` clamp that `SetVolumetricLightmapSettings` warns about.

## Repro

1. `lighting.create_lightmass_volume {location:{...}, size:{x:1000,y:1000,z:1000}, name:"LMIV"}`
   → `success: true`, `existsAfter: true`.
2. `actor.get_bounding_box {actorName:"LMIV"}` → expect `extent:[0,0,0]`.
3. `actor.describe` → expect actor scale `(5,5,5)`, i.e. `size / 200` written onto nothing.
4. `lighting.build_lighting` → completes; `LightingResults` carries **no** "No importance volume
   found" warning, because a volume was counted.

Steps 1-4 are **predicted from source, not measured** — see
[What was NOT done](#what-was-not-done).

## Workaround

Use **`volume.create_lightmass_importance_volume`** (`Handlers/Volume/VolumeHandler.cpp`) instead.
It spawns the same `ALightmassImportanceVolume` through `VolumeHelpers::SpawnVolumeActor`'s `ABrush`
overload, which calls `CreateBoxBrushForVolume(Volume, Extent)` whenever `Extent != FVector::ZeroVector`,
and defaults `extent` to `(5000, 5000, 2000)`. That verb builds real geometry and is unaffected by
this defect.

Two verbs in two namespaces create the same actor class, one correct and one phantom, and the
broken one is the one whose namespace also owns `lighting.build_lighting` — which is the pairing a
caller doing "set up lighting, then bake" will reach for. Note the parameter names differ across
the two (`size` full-size vs `extent` half-extent, `name` vs `volumeName`, and the `lighting` one
takes no `rotation`), so the swap is not mechanical.

To repair a volume already spawned by this verb: `volume.set_volume_extent` (its `ABrush` branch
routes to `CreateBoxBrushForVolume`), **then reset actor scale to `(1,1,1)`** — the `Size/200`
scale survives the repair and multiplies the new geometry. That ordering caveat is
`B-spawned-volumes-have-no-brush-geometry` `#2`/`#3`, and its `volumeName`-vs-`actorName`
parameter obstacle applies here unchanged.

## Fix

Reuse the shared helper the sibling already lifted — do not re-derive it:

- **`VolumeBrushGeometry::BuildBoxBrushGeometry`** in
  `Source/PinWright/Private/Handlers/Volume/VolumeBrushGeometry.h`. `#include` it in
  `LightingHandler.cpp` and call `BuildBoxBrushGeometry(Volume, Size)` after
  `PinWright::MarkLevelActorSpawned`. `BoxSize` is FULL size on each axis, which is already what
  this verb's `size` parameter means (`FVector(1000,1000,1000)` default), so the units line up with
  no conversion — the volume comes out at `location ± Size/2` at identity scale.
- **Delete `Volume->SetActorScale3D(Size / 200.0f)`** in the same edit. Leaving it converts a
  zero-extent volume into one `Size/200` too large per axis; see the section above.
- Change the local `AActor* Volume` to `ALightmassImportanceVolume*` (or `ABrush*`) so the call
  type-checks without a cast — the spawn already names the concrete class.
- The `size` default of `(1000,1000,1000)` is much tighter than
  `volume.create_lightmass_importance_volume`'s `(5000,5000,2000)`. Worth aligning, but that is a
  behaviour choice, not part of this defect.

Consider also making one of the two verbs an alias of the other, or documenting the pairing:
neither appears under a `### lighting.create_lightmass_volume` section in
`Docs/wiki-src/lighting.md` today (the file's only Lightmass mention is about static sky lights),
so a caller has no way to learn which of the two builds geometry.

## Same shape as

- `B-spawned-volumes-have-no-brush-geometry` (IN-REVIEW, High) — **the parent.** Its
  `#4-lifted-brush-builder-to-both-spawn-paths` named this exact site as unfixed and asked for this
  ticket. Its lifted header is the fix here. Read it first.
- `B-blocking-volume-no-brush-geometry` (IN-REVIEW, Critical) — original diagnosis of the brush-init
  mechanism; its `#3-fix-brush-model-init` wrote the builder now living in `VolumeBrushGeometry.h`.
- `B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) — the class statement: the call
  succeeds, every number it reports is correct, and the output is wrong because the deciding number
  was never reported.

## Related

- `E-volume-get-info-zero-extent-ambiguous` (OPEN, Low) — a `brushValid` / `hasGeometry` flag would
  catch this population too, though an importance volume is not what a caller auditing "volumes"
  has in mind.
- `B-lighting-build-lighting-no-completion-signal` (board) — same namespace, and the verb that
  commits the damage. A build that cannot signal completion also cannot report that it baked
  against a point.
- `F-generic-volume-creator-with-brush-geometry` (OPEN, Medium) — a fourth broken `AVolume` spawn
  path is more evidence that per-namespace volume spawning keeps re-introducing this defect.

## What was NOT done

- **No fix was attempted.** No plugin source was modified.
- **Nothing was measured live.** This ticket is a source read of the plugin plus a source read of
  UE 5.8 (`UBrushComponent::CalcBounds`, `FStaticLightingSystem::GatherScene`,
  `FLightmassExporter::AddImportanceVolume` / `SetVolumetricLightmapSettings`, `BaseLightmass.ini`,
  `SceneExport.h`). No RPC was issued and no lighting build was run — an automation suite was live
  on this checkout. The bounds result and the fallback-suppression are both direct reads of
  unconditional engine code, so confidence there is high; the **downstream bake quality** in
  consequence steps 2-3 is inferred from Lightmass's shipped settings and header documentation,
  because the solver's `Programs/UnrealLightmass/Private` sources are not present in this installed
  engine (only `Public` headers ship). A verifier who wants the bake half confirmed should bake a
  small map twice — once with no importance volume, once with a phantom — and diff the result.
- **Only `ALightmassImportanceVolume` was traced.** Other `SpawnActorInActiveWorld` callers were
  grepped for `ABrush` subclasses and only `PostProcessVolumeUtils.h` came back, already cleared by
  `#4`; that grep was by call site, not by class hierarchy, so it is not a proof of exhaustiveness.

severity rationale: impact=High — the README's High band verbatim, "silent false-success ... the caller trusts a result that is a lie and builds on it": the verb returns `success:true` and `existsAfter:true` for a volume whose importance region is a point, and no field in the response could reveal it. NOT Critical: the Critical band names an editor crash or a write that corrupts or loses asset data, and this verb does neither — it writes a valid actor with a valid transform and an empty brush, and deleting it loses nothing. The `MapBuildData` overwrite that *does* destroy a prior good bake is committed by `lighting.build_lighting`, a separate verb the user chose to run, so it is downstream damage rather than this write; a fixer who disagrees should re-rate rather than assume. The aggravating factor that keeps this at the TOP of High rather than lower is that the phantom is worse than the absence: `FStaticLightingSystem::GatherScene` gates its automatic scene-bounds fallback on `GetImportanceVolumes().Num() == 0`, so calling this verb SUPPRESSES a correct auto-synthesized importance region and silences the warning that would have told the user, leaving a level that would have baked correctly baking direct-only. × reach: both modifiers declined. Not up — one verb of thirteen in `lighting.*`, and only sessions that bake static lighting reach the consequence. Not down — this is not a "rare edge path": it is the standard way to prepare a level for a static lighting bake, it sits in the same namespace as `lighting.build_lighting`, and the two are the natural pair for a level-lighting task. High stands unmodified, and matches the parent ticket `B-spawned-volumes-have-no-brush-geometry` for the same mechanism.

## History
- `#1-fourth-phantom-volume-site-still-open` `OPEN` reporter — Filed at the request of `B-spawned-volumes-have-no-brush-geometry` `#4-lifted-brush-builder-to-both-spawn-paths`, which named this site as a deliberately-unfixed fourth instance. **Verified still present at HEAD by source read; nothing measured live** (an automation suite was running on this checkout, so no RPC was issued and no bake was run). **Name correction to that entry:** the verb is registered `lighting.create_lightmass_volume`, not `lighting.create_lightmass_importance_volume` — the latter is a *different and working* verb in `volume.*`. **Mechanism:** `LightingHandler.cpp`'s `lighting.create_lightmass_volume` body raw-spawns via `SpawnActorInActiveWorld<AActor>(ALightmassImportanceVolume::StaticClass(), ...)` then writes `SetActorScale3D(Size / 200.0f)`; the file contains zero occurrences of `Brush` / `UModel` / `UPolys`. `ALightmassImportanceVolume : public AVolume`, so `UBrushComponent::CalcBounds` falls to its final `else` and returns `FBoxSphereBounds(LocalToWorld.GetLocation(), FVector::ZeroVector, 0.f)` — a literal zero extent that no scale can multiply, which is why the scale write is inert today. `AddActorVerification` emits no bounds field, so `success:true` + `existsAfter:true` is all the caller sees. **Sibling's fix does NOT cover it:** the `#4` arm `Cast<AVolume>(Spawned)` → `BuildBoxBrushGeometry` lives inside `SpawnHandler.cpp`'s `actor.spawn` handler body, after that file's own local `SpawnActorInWorld`; this verb enters through the unrelated `SpawnActorInActiveWorld<T>` template in `Utils/AssetUtils.h`, where `grep "Brush\|AVolume"` returns nothing. Different layer, never meet. **Consequence, and it is worse than the generic phantom:** `FStaticLightingSystem::GatherScene` (Engine `StaticLightingSystem.cpp`) synthesizes a scene-bounds importance volume only `if (LightmassExporter->GetImportanceVolumes().Num() == 0)` — a COUNT, not an extent — so the phantom counts as 1, the fallback is skipped, and the "No importance volume found" message-log warning inside that branch never fires. A level that would have baked correctly with NO volume now bakes against a point. `FLightmassExporter::AddImportanceVolume` stores `GetComponentsBoundingBox(true)` unchecked; `SetVolumetricLightmapSettings` then gets `ImportanceExtent (0,0,0)` → `FullGridSize (1,1,1)` → at stock `BrickSize=4`/`MaxRefinementLevels=3`/`DetailCellSize=200` a volumetric lightmap of one 12800 uu cube whose MIN corner is the spawn point; and `BaseLightmass.ini`'s `bEmitPhotonsOutsideImportanceVolume=False` means, per `SceneExport.h`, "nothing outside the volume will bounce lighting and will be lit with direct lighting only" — the whole level being outside. **Scale check (question 3): same matched pair as the foliage verb.** The `Size/200` write is inert now but multiplies correct geometry after a naive fix — `Size/200` overshoot per axis, 5x at the verb's own `(1000,1000,1000)` default, 100x at level scale. It must be deleted in the same edit, and any migration of already-spawned volumes must normalize scale to `(1,1,1)`. **Fix:** `#include` and call `VolumeBrushGeometry::BuildBoxBrushGeometry(Volume, Size)` from `Handlers/Volume/VolumeBrushGeometry.h` — its `BoxSize` is FULL size, matching this verb's `size` with no conversion. **Workaround:** `volume.create_lightmass_importance_volume`, which routes through `SpawnVolumeActor`'s `ABrush` overload → `CreateBoxBrushForVolume` and builds real geometry (default extent `5000,5000,2000`); parameter names differ (`extent` half-extent vs `size` full, `volumeName` vs `name`), so the swap is not mechanical. Dedup: board grepped for `lightmass`, `importance volume` and `brush geometry` — only `B-spawned-volumes-have-no-brush-geometry`, `E-volume-get-info-zero-extent-ambiguous` and `F-generic-volume-creator-with-brush-geometry` matched, none owning this verb, and no `lighting.*` ticket covers it. NOT DONE: no source modified; the bounds result and fallback suppression are direct reads of unconditional engine code, but the downstream bake quality is inferred from shipped Lightmass settings and headers because the solver's `Programs/UnrealLightmass/Private` sources are not in this installed engine.
- `#2-brush-geometry-with-measured-bounds-and-a-refusal` `IN-REVIEW` developer — "Confirmed the defect at HEAD in `unreal-fpv-new` before touching anything: `LightingHandler.cpp` raw-spawned `ALightmassImportanceVolume` and wrote `SetActorScale3D(Size / 200.0f)`, and `grep -c 'Brush|UModel|UPolys'` over the file was **0**. Re-derived every engine citation in this checkout's UE 5.8 rather than trusting the ticket's line numbers, and **the count-gating claim holds**: `StaticLightingSystem.cpp:2340` is `if (LightmassExporter->GetImportanceVolumes().Num() == 0)` with the synthesis + `LightmassError_MissingImportanceVolume` PerformanceWarning inside that branch, the `TObjectIterator<ALightmassImportanceVolume>` loop above it (`:2277-2282`) applies no extent test, `Lightmass.h:93-96` stores `GetComponentsBoundingBox(true)` unchecked, and `BrushComponent.cpp:459-484` returns a literal `FVector::ZeroVector` extent on the null-brush path. So High stands and the fix shape is unchanged. **Fix:** included the existing shared `Handlers/Volume/VolumeBrushGeometry.h` and call `BuildBoxBrushGeometry(Volume, Size)` — no second builder — retyped the local to `ALightmassImportanceVolume*`, and **deleted the `Size / 200` scale write** in the same edit (it is inert on a null brush but overshoots `Size/200` per axis once geometry exists). **Response honesty:** the reply now carries `measuredSize` / `measuredExtent` / `measuredCenter` read back off the actor via `AVolume::GetBounds()` (the same `UBrushComponent::CalcBounds` behind the cached bounds the exporter reads), with `requestedSize` / `requestedLocation` named separately and a `sizeWarning` when they disagree by >1 uu — a zero-extent volume is now detectable from the response alone, which it was not before (the old reply had no bounds field of any kind). **Refusal:** a non-finite or non-positive `size` is refused with `INVALID_ARGUMENT` before the spawn, and a brush that fails to build or measures zero extent causes the spawned actor to be **destroyed** and the verb to return `SPAWN_FAILED`; both messages spell out that the engine's fallback is gated on the volume COUNT, so a spawned-but-empty volume is worse than none. **Sibling check (tag `spawn-path`):** the only other volume spawn in `LightingHandler.cpp` is `FindOrSpawnUnboundPPV` (`PostProcessVolumeUtils.h`), which sets `bUnbound = true`; verified in-engine that `APostProcessVolume::GetProperties` then reports `Size = DBL_MAX` and skips the bounds test, so it is not the same defect and was left alone. No other lighting verb spawns an `ABrush`. **Regression tests** added to the parent ticket's own file `Tests/World/TestSpawnedVolumeBrushGeometry.cpp`, reusing its `SpawnedVolumeBrushGeometryTestHelpers`: `PinWright.lighting.create_lightmass_volume.VolumeBuildsBrushGeometry` spawns at `size (3000, 2000, 800)` (deliberately not the cube default, so an echoed field cannot pass as a measured one) and asserts brush model + polys, identity actor scale, bounds half-extent == size/2 centred on the request, `EncompassesPoint` containment, and that `measuredSize`/`measuredExtent` in the reply match the actor; `...DegenerateSizeIsRefusedNotSpawned` sends `z: 0` and asserts the call fails, the message explains the count gate, and the `ALightmassImportanceVolume` count in the level is unchanged. Both fail today for the right reasons (no brush, scale 15/10/4, no bounds field; and success + one leftover volume). `check_test_ids.py` re-run: CLEAN, 4765 ids, no dot-prefix collisions. Also appended a `### lighting.create_lightmass_volume` section to `Docs/wiki-src/lighting.md` (after the last existing `###`, so no `##` section is orphaned) documenting the two-verbs-one-class trap and the differing parameter shapes the ticket flagged. NOT DONE: not compiled and no suite run — the orchestrator builds; and the downstream bake-quality half of the ticket (consequence steps 2-3) is still unmeasured, since Lightmass's solver sources do not ship with the installed engine."
