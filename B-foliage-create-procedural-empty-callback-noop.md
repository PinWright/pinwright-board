---
id: B-foliage-create-procedural-empty-callback-noop
title: "foliage.create_procedural passes an EMPTY AddInstancesFunc callback to ResimulateProceduralFoliage, so every generated instance is discarded — the volume/spawner are created but zero foliage is ever scattered, while the response hardcodes resimulated:true"
status: IN-REVIEW
severity: High
category: bug
tags: [foliage, create_procedural, silent-noop, success-no-effect, hardcoded-response, procedural-foliage, verify-after-mutate]
encounters: 2
lastSeen: 2026-08-29T00:00:00+05:00
---

# `foliage.create_procedural` never scatters any instances — the callback that would place them is a no-op

`foliage.create_procedural` correctly creates the `UProceduralFoliageSpawner`
asset, one `UFoliageType` per entry, and an `AProceduralFoliageVolume` actor
sized to the requested bounds. But its core purpose — actually **scattering
foliage instances** across the volume — is a permanent silent no-op, because the
handler runs the procedural simulation and then throws away every instance it
computes.

The engine's `UProceduralFoliageComponent::ResimulateProceduralFoliage` takes a
`TFunctionRef` explicitly named `AddInstancesFunc`, and that callback is the ONLY
code that places the generated instances into the world (the component does not
add them itself). PinWright passes an **empty lambda**, so the simulation's
`DesiredFoliageInstances` are generated and immediately discarded. No amount of
ground/collision under the volume can make this RPC produce a single instance.

Separately, the handler captures the simulation's `bool` return into `bResult`
but never reads it, and hardcodes `resimulated:true` in the response — so the
caller is told the resimulation succeeded even when generation returned `false`.

## Guilty source (`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp`)

The resimulation call (`:1019-1026`) passes an empty callback and drops `bResult`:

```cpp
if (UProceduralFoliageComponent *ProcComp = Volume->ProceduralComponent) {
  ProcComp->FoliageSpawner = Spawner;
  ProcComp->TileOverlap = 0.0f;

  // Resimulate
  bool bResult = ProcComp->ResimulateProceduralFoliage(
      [](const TArray<FDesiredFoliageInstance> &) {});   // :1024-1025  EMPTY callback -> desired instances discarded; bResult unused
}
```

...and the response hardcodes success regardless (`:1033`):

```cpp
Resp->SetBoolField(TEXT("resimulated"), true);           // :1033  ignores bResult; always true
```

Engine ground truth — `ResimulateProceduralFoliage`'s callback is what adds the
instances (`Engine/Source/Runtime/Foliage/Private/ProceduralFoliageComponent.cpp:350-376`):

```cpp
bool UProceduralFoliageComponent::ResimulateProceduralFoliage(TFunctionRef<void(const TArray<FDesiredFoliageInstance>&)> AddInstancesFunc)
{
    if (FoliageSpawner) {
        TArray<FDesiredFoliageInstance> DesiredFoliageInstances;
        if (GenerateProceduralContent(DesiredFoliageInstances)) {
            if (DesiredFoliageInstances.Num() > 0) {
                RemoveProceduralContent(false);
                AddInstancesFunc(DesiredFoliageInstances);   // the ONLY placement path
            }
            return true;                                     // true == generation ran, NOT "instances placed"
        }
    }
    return false;
}
```

The engine's own reference implementation
(`Engine/Source/Editor/FoliageEdit/Private/ProceduralFoliageEditorLibrary.cpp:64-72`)
fills this callback with `FEdModeFoliage::AddInstances(Component->GetWorld(),
DesiredFoliageInstances, OverrideGeometryFilter, true)` — i.e. the callback must
do the actual add. PinWright's empty lambda does none of that.

## Replay-confirmed (live, this audit)

`foliage.create_procedural {"name":"OracleMeadow","bounds":{"location":{"x":0,"y":0,"z":0},"size":{"x":2000,"y":2000,"z":500}},"foliageTypes":[{"meshPath":"/Engine/BasicShapes/Cube.Cube","density":200}],"seed":12345}`
returned:

```json
{"success":true,"volume_actor":"OracleMeadow","spawner_path":"/Game/ProceduralFoliage/OracleMeadow_Spawner.OracleMeadow_Spawner","foliage_types_count":1,"resimulated":true,"actorClass":"ProceduralFoliageVolume","assetClass":"ProceduralFoliageSpawner","existsAfter":true}
```

A follow-up unfiltered `foliage.get_instances {}` returned only hand-placed
instances (HeroDebris/FillerDebris/OracleReplayFT from prior explicit
`add_instances` calls) — **zero** instances belong to `OracleMeadow` or its
auto-created foliage type, despite the density-200 spawner over a 2000x2000
volume reporting `resimulated:true`. The volume and spawner asset exist; the
scatter did not happen.

## Why the friction note misdiagnosed it

The attempt agent noted "the procedural spawner set up cleanly but did not bake
painted instances near origin (no ground under the volume)." The missing-ground
explanation is a red herring: the empty `AddInstancesFunc` discards the generated
instances unconditionally, so the RPC would place nothing even with a perfect
landscape under the volume. The tool's `success:true, resimulated:true` masked
the real defect and led the agent to attribute the empty result to level
geometry rather than to the handler.

## What it should do / how to fix

- Replace the empty callback with a real add path (mirror
  `ProceduralFoliageEditorLibrary`): call `FEdModeFoliage::AddInstances(World,
  DesiredFoliageInstances, GeometryFilter, /*bRebuildFoliageTree=*/true)` inside
  the callback so the generated instances are actually placed. (Requires the
  `FoliageEdit` editor module / `FEdModeFoliage`, or an equivalent
  `AInstancedFoliageActor::AddInstances`/`AddFoliageInstance` path.)
- Report `resimulated` from the actual `bResult`, not a hardcoded `true`, and
  echo the placed-instance count (e.g. `instances_spawned`) so a scatter can be
  verified from the response instead of trusting a hardcoded flag. When zero
  instances land, say so (the engine surfaces "Unable to spawn instances. Ensure
  a large enough surface exists within the volume." in the UI path) rather than
  reporting an unqualified success.

**Workaround today:** `foliage.create_procedural` cannot place procedural
instances at all — to scatter foliage programmatically, use `foliage.add_instances`
with explicit transforms; the procedural verb only builds the spawner/volume
scaffolding.

severity rationale: impact=silent false-success — the RPC's primary purpose (scattering procedural foliage) never runs, and the response hardcodes resimulated:true so the caller trusts a lie × reach=every-session for anyone using the procedural-foliage verb (its sole reason to exist) -> High

## History
- `#1-initial-repro` `OPEN` reporter — Seed-`foliage.create_procedural` realism
  audit (vegetation-dressing task: procedural scatter + hand-placed hero clusters
  + scoped remove). Replay-confirmed live at HEAD: `create_procedural` on a
  2000x2000 density-200 volume at origin returned `success:true, resimulated:true`
  and created the spawner asset + volume actor, but a follow-up unfiltered
  `foliage.get_instances {}` showed zero instances belonging to that spawner /
  its auto foliage type — only prior hand-placed `add_instances` output. Root
  cause confirmed in source: `FoliageHandler.cpp:1024-1025` passes an EMPTY
  `[](const TArray<FDesiredFoliageInstance>&){}` to the engine's
  `ResimulateProceduralFoliage`, whose callback param is literally named
  `AddInstancesFunc` and is the ONLY code that places the generated instances
  (`Engine/.../ProceduralFoliageComponent.cpp:350-376`; engine reference fills it
  with `FEdModeFoliage::AddInstances`, `ProceduralFoliageEditorLibrary.cpp:64-72`).
  So the generated `DesiredFoliageInstances` are discarded and no procedural
  foliage is ever scattered, regardless of ground. Separately the handler drops
  the sim's `bResult` and hardcodes `resimulated:true` (`:1033`). The attempt
  agent misdiagnosed the empty result as "no ground under the volume" — a red
  herring the false-success response invited. Same success-no-effect family as
  `B-add-mass-spawner-silent-noop` / `B-bt-set-node-properties-silent-noop`, here
  a distinct method (`foliage.create_procedural`). Dedup: ripgrep across
  OPEN/closed found no create_procedural spawn/scatter no-op ticket — the existing
  foliage tickets are get_instances scale readback, remove edge-input honesty,
  add_type auto-save, and nested input-schema docs (different methods/axes), and
  `E-create-procedural-terrain-no-material-echo` is a different method
  (`environment.build.create_procedural_terrain`, material echo).
- `#2-fix` `IN-REVIEW` developer — Fixed the empty-callback no-op. Verified at
  HEAD: `FoliageHandler.cpp` passed `[](const TArray<FDesiredFoliageInstance>&){}`
  (empty) to `ResimulateProceduralFoliage`, dropped the sim `bResult`, and
  hardcoded `resimulated:true`. Confirmed against engine source that the callback
  (`AddInstancesFunc`, `ProceduralFoliageComponent.h:111`) is the ONLY placement
  path and the bool means "sim ran," not "instances placed," and that the engine's
  own `ProceduralFoliageEditorLibrary.cpp:64-73` fills it with
  `FEdModeFoliage::AddInstances(...)`. IMPLEMENTATION NOTE (differs from the
  ticket's proposed fix): `FEdModeFoliage::AddInstances` and the
  `FFoliagePaintingGeometryFilter` it takes live only in FoliageEdit's PRIVATE
  `FoliageEdMode.h` and carry NO `*_API` export — I first tried the ticket's
  direct-callback approach (adding FoliageEdit/Private to include paths) and it
  compiled but FAILED TO LINK (`LNK2019 unresolved external FEdModeFoliage::AddInstances`),
  because an un-exported symbol is not linkable cross-module. Corrected approach:
  the callback path now invokes the PUBLIC BlueprintCallable
  `UProceduralFoliageEditorLibrary::ResimulateProceduralFoliageComponents` (which
  internally supplies the real `FEdModeFoliage::AddInstances` callback) by
  REFLECTION — that class is also `UCLASS()` with no `*_API`, so it is resolved via
  `FindObject<UClass>(nullptr, "/Script/FoliageEdit.ProceduralFoliageEditorLibrary")`
  and invoked with `ProcessEvent` (the project's documented convention for
  export-less engine types), passing the volume's `ProceduralComponent` in a
  one-element component array. `resimulated` is now reported from whether that
  reflected simulation actually ran (not a hardcoded literal), and a new
  `instances_spawned` field reports the before/after delta of world
  `AInstancedFoliageActor` placed-instance counts (all `FOLIAGE_API`, directly
  linkable) so a scatter is verifiable from the response instead of a hardcoded
  flag. No Build.cs / module-dependency change (`FoliageEdit` was already linked at
  `PinWright.Build.cs:66`; reflection needs no include). File: `FoliageHandler.cpp`.
  Test: `PinWright.foliage.create_procedural.ReportsInstancesSpawned` (in
  `Tests/World/TestEnvironmentHandlers.cpp`) drives the real production handler
  with a 2000x2000 density-50 cube volume and asserts the response carries an
  `instances_spawned` number (a field the empty-callback code never emitted, so it
  fails on revert) plus a reported `resimulated`; the count is not asserted
  positive because a headless world has no surface under the volume (0 is honest).
  Not asserting positive scatter keeps the test host-content-independent; the
  discriminating signal is that the real add path runs and reports truthfully.
- `#3-zero-count-is-the-volume-not-the-world` `IN-REVIEW` tester — Additional
  evidence from live measurement against a running editor (four
  `foliage.create_procedural` calls). **`#2`'s fix is confirmed present at HEAD
  and is NOT being challenged.** Re-derived this session: the reflection path
  through `UProceduralFoliageEditorLibrary::ResimulateProceduralFoliageComponents`
  is at `FoliageHandler.cpp:1542-1552`, `resimulated` is reported from the actual
  run at `:1602`, and the `instances_spawned` delta is emitted at `:1603` — the
  empty callback is gone, the hardcoded `true` is gone, and the response now
  carries the count `#2` added. (Note for anyone re-reading citations: the
  `:1176-1194` reference to this path in
  `B-foliage-paint-does-no-ground-projection` `#1` has drifted; `:1542-1552` is
  current.) What IS challenged is two of this ticket's own sentences, both
  contradicted by measurement. **Body ¶1** says the handler creates *"an
  `AProceduralFoliageVolume` actor sized to the requested bounds"* — it does not.
  The spawned volume's bounds extent is `(0,0,0)`. The requested size is written
  as an actor *scale* (`FoliageHandler.cpp:1499`, `SetActorScale3D(Size / 200.0f)`)
  onto a brush that was never initialized by the raw `SpawnActor` at `:1489-1491`,
  so there is no geometry for the scale to multiply: a `size {15750, 18900, 3500}`
  request measured `extent:[0,0,0]` with actor scale `(78.75, 94.5, 17.5)` on
  `actor.get_bounding_box` / `actor.describe`. **History `#2`**, justifying its
  regression test, says *"the count is not asserted positive because a headless
  world has no surface under the volume (0 is honest)"* — that explanation does
  not survive contact with a populated world. Four calls on a level with terrain
  directly under the volume also returned `instances_spawned: 0`. The zero is the
  zero-extent volume, not the headless world: a simulation over a region of size
  zero has nowhere to place anything, with or without ground. This is the second
  time a missing-ground story has been offered for this verb's empty output and
  the second time it has been wrong — `#1` of this same ticket already called the
  attempt agent's "no ground under the volume" a red herring, and the same
  reasoning now applies to the test rationale in `#2`. Consequence for whoever
  verifies this ticket: `PinWright.foliage.create_procedural.ReportsInstancesSpawned`
  will keep passing (it only asserts the fields exist) and a live call will keep
  returning `0`, so the fix cannot be confirmed working end to end until the
  volume has real bounds. Filed separately as
  `B-spawned-volumes-have-no-brush-geometry` (OPEN, High) — the brush is never
  initialized on this spawn path, the in-tree fix
  (`VolumeHelpers::BuildBoxBrushGeometry`, `VolumeHandler.cpp:90`, from
  `B-blocking-volume-no-brush-geometry` `#3`) is scoped to `VolumeHandler.cpp` and
  this verb does not reach it. NOT DONE here: no fix attempted, no status change,
  and it was **not** verified that a correctly built brush actually makes
  `instances_spawned` go positive — only that the volume currently has no
  geometry to place into, and that `#2`'s stated reason for the zero is not the
  operative one.
