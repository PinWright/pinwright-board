---
id: B-blocking-volume-no-brush-geometry
title: "volume.create_blocking_volume reports success but builds a null-brush, zero-collision volume"
status: IN-REVIEW
severity: Critical
category: bug
tags: [volume, brush, blocking-volume, silent-noop, collision]
---

# volume.create_blocking_volume reports success but builds a null-brush, zero-collision volume

`volume.create_blocking_volume` (and the two follow-up editors `volume.set_volume_extent`
and `volume.set_volume_bounds`) spawn / report success but produce a `BlockingVolume`
whose brush is **null** — it has no geometry and therefore no collision. The actor lands
at the requested location, but the `extent` is silently dropped: the volume is a
zero-size phantom. This defeats the entire purpose of a blocking volume (invisible
collision walls), and the tool gives no indication anything went wrong — a
silent success-with-no-effect.

## What's wrong

The created volume's `BrushComponent0.Brush` is `null` and its world bounding box is
`extent [0,0,0]` regardless of the `extent` passed to create, or the values passed to
`set_volume_extent` / `set_volume_bounds` afterward. All three calls return a success
payload (create returns `existsAfter:true`; `set_volume_extent` echoes
`newExtent:{...}`; `set_volume_bounds` echoes `bounds:{min,max}`), but none of them
actually changes the volume's geometry. `volume.get_volumes_info` then reports every
brush volume with `extent:{0,0,0}` — not (as one might assume) a reporting quirk in
`get_volumes_info`, but the ground truth: the brushes really are empty.

## Root cause

`CreateBoxBrushForVolume` in
`Source/EditorAutomationRpcGateway/Private/Handlers/Volume/VolumeHandler.cpp:77`:

```cpp
bool CreateBoxBrushForVolume(ABrush* Volume, const FVector& Extent)
{
    if (!Volume) return false;
    UCubeBuilder* CubeBuilder = NewObject<UCubeBuilder>(GetTransientPackage());
    CubeBuilder->X = Extent.X * 2.0f;
    CubeBuilder->Y = Extent.Y * 2.0f;
    CubeBuilder->Z = Extent.Z * 2.0f;
    CubeBuilder->Build(Volume->GetWorld(), Volume);   // <-- no UModel to write into
    return true;
}
```

A freshly `World->SpawnActor<ABlockingVolume>()` brush actor has a **null `Brush` UModel**.
`UCubeBuilder::Build` has no initialized model on the target volume to populate, so the
geometry goes nowhere and the volume stays empty. The engine's own brush-actor creation
path (`Engine/.../Factories/ActorFactory.cpp`) initializes the model first:

```cpp
NewActor->Brush = NewObject<UModel>(NewActor, NAME_None, ObjectFlags);
NewActor->Brush->Initialize(nullptr, true);
NewActor->Brush->Polys = NewObject<UPolys>(NewActor->Brush, NAME_None, ObjectFlags);
NewActor->GetBrushComponent()->Brush = NewActor->Brush;
```

`CreateBoxBrushForVolume` skips this, so there is no model for the cube builder to fill.
The same helper backs `CreateSphereBrushForVolume` / `CreateCapsuleBrushForVolume` and is
reused by `set_volume_extent` (line ~1305) and `set_volume_bounds` (line ~1440), so all of
them are affected. The validation gate `ValidateExtent` even rejects non-positive extents,
reinforcing the impression that a positive extent will be applied — but it never is.

## What it should do

Initialize the volume's `Brush` UModel (and `Polys`, and wire `BrushComponent->Brush`)
before invoking the cube builder, then call `Volume->Brush->Modify()` / rebuild bounds so
the resulting brush has real geometry and collision. After a successful create, the
volume's `BrushComponent0.Brush` must be non-null and `actor.get_bounding_box` must return
the requested half-extent.

## Verbatim repro

1. `volume.create_blocking_volume`
   args: `{"volumeName":"Probe_Vol_A","location":{"x":5000,"y":5000,"z":100},"extent":{"x":300,"y":400,"z":500}}`
   -> success: `{"volumeName":"Probe_Vol_A","volumeClass":"ABlockingVolume","actorName":"Probe_Vol_A","existsAfter":true,"actorClass":"BlockingVolume", ...}` (note: no extent echoed).
2. `actor.get_bounding_box` args `{"actorName":"Probe_Vol_A"}`
   -> `{"origin":[5000,5000,100],"extent":[0,0,0]}`  (should be ~[300,400,500]).
3. `actor.get_component_property` args `{"actorName":"Probe_Vol_A","componentName":"BrushComponent0","propertyName":"Brush"}`
   -> `{"value":null}`  (brush model never created).
4. `volume.set_volume_extent` args `{"volumeName":"Probe_Vol_A","extent":{"x":300,"y":400,"z":500}}`
   -> success echo `{"newExtent":{"x":300,"y":400,"z":500}, ...}` but
   `actor.get_bounding_box` still `{"extent":[0,0,0]}`.
5. `volume.set_volume_bounds` args `{"volumeName":"Probe_Vol_A","bounds":[4700,4600,-400,5300,5400,600]}`
   -> success echo `{"bounds":{"min":[4700,4600,-400],"max":[5300,5400,600]}, ...}` but
   `actor.get_bounding_box` still `{"extent":[0,0,0]}`.
6. `volume.get_volumes_info` args `{"filter":"Probe_Vol_A"}`
   -> `{"volumes":[{"name":"Probe_Vol_A","class":"BlockingVolume","location":{"x":5000,"y":5000,"z":100},"extent":{"x":0,"y":0,"z":0}}]}`.

Impact: any task that places blocking volumes / kill-Z volumes / trigger box brushes via
these RPCs gets actors with no collision geometry, while every call reports success — the
failure is invisible until something walks through the "wall." Critical because the
capability appears to work and is silently broken.

## History
- `#1-initial-repro` `OPEN` reporter — Found via the arena-walls task (ring of 5 blocking volumes). Replay-confirmed live on UE 5.7 host: created `Probe_Vol_A` with `extent {300,400,500}`; `actor.get_bounding_box` returns `[0,0,0]`, `BrushComponent0.Brush` is `null`, and `get_volumes_info` reports `extent {0,0,0}`. `set_volume_extent` and `set_volume_bounds` echo success but leave the bbox at `[0,0,0]`. Root cause: `CreateBoxBrushForVolume` (VolumeHandler.cpp:77) runs `UCubeBuilder::Build` against a volume whose `Brush` UModel was never initialized (cf. engine `ActorFactory.cpp` which creates the UModel/Polys first). Seed method `volume.create_blocking_volume` is the primary culprit; the two setters share the same broken helper.
- `#2-additional-trigger-box-resize` `OPEN` reporter — Additional evidence (seed `volume.set_volume_bounds`, arena gameplay-volumes task): the silent-resize-failure is NOT limited to `BlockingVolume`/`extent {0,0,0}` — it also breaks `TriggerBox` (`ATriggerBox`), and the readback extent is non-zero-but-wrong rather than `{0,0,0}`, which makes the no-op even harder to spot. Replay on UE 5.7 host: created fresh `ReplayBoundsTest` (TriggerBox, `boxExtent {100,100,100}`); `volume.set_volume_bounds {bounds:[-300,-300,0,300,300,400]}` returned `{bounds:{min:[-300,-300,0],max:[300,300,400]},center:{0,0,200}}` (looks correct), but `get_volumes_info` readback = `location {0,0,200}` (center applied) with `extent {128,128,128}` — NOT the requested half-extent `{300,300,200}`. Re-ran `set_volume_bounds {bounds:[-500,-500,-500,500,500,500]}` -> readback `extent {200,200,200}`, not `{500,500,500}`. `set_volume_extent {extent:{350,350,450}}` echoed `newExtent {350,350,450}` but readback = `{140,140,180}`. Mechanism confirmed in source: `ATriggerBox` is an `ABrush`, so both setters take the `Cast<ABrush>` branch and call the broken `CreateBoxBrushForVolume` (set_volume_bounds VolumeHandler.cpp:1440, set_volume_extent :1305) — they never reach the `SetActorScale3D` fallback (:1444/:1309) that non-brush volumes use. `SetActorLocation(Center)` IS applied, so the center round-trips and an agent confirming only the center believes "round-trip confirmed" while the extent silently failed (exactly what the attempt agent self-reported). Same root cause as #1; widens confirmed scope to TriggerBox and documents the misleading non-zero readback.
- `#4-additional-create-trigger-box-extent-dropped` `IN-REVIEW` reporter — Additional evidence (arena gameplay-triggers task, seed `volume.remove_volume`): the silent-extent-drop is present at **create time on `create_trigger_box`**, not only on the setters. The existing #2 created `boxExtent {100,100,100}` (the engine default) and exercised only `set_volume_bounds`/`set_volume_extent`; this replay shows a *non-default* `boxExtent` passed to `create_trigger_box` is itself dropped before any setter runs. Replay-confirmed live on UE 5.7 host (fix #3 not yet compiled/deployed here, so still reproduces as expected): `volume.create_trigger_box {volumeName:"ReplayExtentTest_Box", location:{2000,2000,100}, boxExtent:{300,300,200}}` returned `{boxExtent:{300,300,200}, existsAfter:true, actorClass:"TriggerBox"}` (echoes the requested extent), but `actor.get_bounding_box` ground truth = `{origin:[2000,2000,100], extent:[128,128,128]}` — the default 128 cube, requested `{300,300,200}` never applied; `get_volumes_info {filter:"ReplayExtentTest_Box"}` correspondingly reports `extent {128,128,128}`. Then `volume.set_volume_extent {extent:{500,500,300}}` echoed `newExtent {500,500,300}` but `actor.get_bounding_box` = `[200,200,128]` (partial-scale, not the requested half-extent — same misleading non-zero readback as #2). `volume.remove_volume` cleaned it up correctly (`existsAfter:false`), so the seed verb is fine — the culprit is the shared `CreateBoxBrushForVolume` brush helper on `create_trigger_box`/`set_volume_extent`. Confirms the fix in #3 must cover the create path (`SpawnVolumeActor`→`CreateBoxBrushForVolume`, :273), not just the setters, for `create_trigger_box` to honor a non-default `boxExtent`.
- `#5-additional-set-extent-scale-confirmed` `OPEN` reporter — Additional evidence (arena gameplay-volumes task, seed `volume.set_volume_extent`). Replay-confirms the TriggerBox round-trip break from #2 with the task's actual values and pins the scale ground truth via `actor.describe` (not just `get_volumes_info`). Live on UE 5.7 fuzz host (fix #3 still not compiled/deployed here, so reproduces as expected): `volume.create_trigger_box {volumeName:"ReplayEntryTrigger", location:{0,0,100}, boxExtent:{200,200,150}}` echoed `boxExtent {200,200,150}` but `get_volumes_info` immediately read back `extent {128,128,128}` (the default cube — create-time extent dropped, matching #4). Then `volume.set_volume_extent {extent:{450,450,250}}` echoed `newExtent {450,450,250}` (its own echo always looks correct — it never re-reads), but `get_volumes_info` read back `extent {180,180,128}` (X/Y partial-scale, Z pinned at the 128 billboard bound — same misleading non-zero readback as #2/#4). `actor.describe ReplayEntryTrigger` confirms the actor scale was written to `{4.5,4.5,2.5}` (= requested extent/100), yet `GetActorBounds()` (the canonical readback in `get_volumes_info` :1581) reports `{180,180,128}` because the brush collision half-extent + editor Sprite billboard never equal extent×scale — so the setter's echo and the canonical readback can never agree for `ATriggerBox`. Same root cause and culprit (`CreateBoxBrushForVolume` shared by `set_volume_extent` :1314 / `create_trigger_box` via `SpawnVolumeActor` :282); no new bug. Confirms fix #3 must make `set_volume_extent`'s echo reflect the post-rebuild `GetActorBounds()` half-extent (or the brush builder's true half-extent), not the raw request, so the round-trip is verifiable.
- `#6-additional-trigger-capsule-set-extent-noop` `OPEN` reporter — Additional evidence (checkpoint gameplay-triggers task, seed `volume.create_trigger_capsule`): the silent set_volume_extent no-op + misleading non-zero readback (documented for `BlockingVolume` in #1 and `TriggerBox` in #2/#4/#5) also affects **`TriggerCapsule`** (`ATriggerCapsule`, created via `volume.create_trigger_capsule`) — the one brush class whose readback was not previously pinned. Replay-confirmed live on UE 5.7 fuzz host (fix #3 not compiled/deployed here, so still reproduces): `Checkpoint_PrimaryTrigger` (TriggerCapsule, created `capsuleRadius 120`/`capsuleHalfHeight 220`) reads back `extent {180,180,572}` in `get_volumes_info`. `volume.set_volume_extent {volumeName:"Checkpoint_PrimaryTrigger", extent:{150,150,260}}` returned `{actorClass:"TriggerCapsule", newExtent:{150,150,260}, existsAfter:true}` (its echo always reflects the raw request), but an immediate `get_volumes_info {filter:"Checkpoint_PrimaryTrigger"}` read back `extent {180,180,572}` — completely UNCHANGED, the requested `{150,150,260}` never applied. Same task also re-confirms the TriggerBox scope from #2: `volume.set_volume_extent {volumeName:"Checkpoint_ExitTrigger", extent:{200,75,50}}` echoed `newExtent {200,75,50}` but `get_volumes_info` read back the unchanged default `extent {128,128,128}`. Mechanism = same shared helper: `ATriggerCapsule` is an `ABrush`, so `set_volume_extent` takes the `Cast<ABrush>` → `CreateBoxBrushForVolume` branch (`CreateCapsuleBrushForVolume` likewise routes through the broken `BuildBoxBrushGeometry`/null-`Brush` path), never the `SetActorScale3D` non-brush fallback. The agent self-reported "enlarged primary to {180,180,572}, larger than original" and believed the resize succeeded — the misleading non-zero readback (#2/#5) fooled the confirmation exactly as predicted. No new bug; widens confirmed scope to `TriggerCapsule` and confirms fix #3 must cover `CreateCapsuleBrushForVolume` / make `set_volume_extent`'s echo reflect the post-rebuild `GetActorBounds()` half-extent for capsule triggers too.
- `#7-additional-trigger-sphere-extent-echo-mismatch` `IN-REVIEW` reporter — Additional evidence (arena gameplay-triggers task, seed `volume.create_trigger_sphere`): widens the `set_volume_extent` echo/round-trip mismatch (#5/#6's "echo always reflects the raw request, never re-reads") to the **non-brush** branch — `TriggerSphere` (`ATriggerSphere`, an `ATriggerBase`, NOT an `ABrush`), the one volume class the prior history never pinned. Distinct from #1/#2/#4/#5/#6 (all brush classes that take the `Cast<ABrush>`→`CreateBoxBrushForVolume` branch where the resize is a silent NO-OP): for the sphere the resize **genuinely takes effect** via the `SetActorScale3D` non-brush fallback (VolumeHandler.cpp:1316-1319), so the ONLY defect here is the echo — `newExtent` is misreported, not the geometry. Replay-confirmed live on UE 5.7 fuzz host (fix #3 not compiled/deployed here): `volume.create_trigger_sphere {volumeName:"ReplayCenterTrigger", location:{0,0,200}, sphereRadius:350}` → `{volumeClass:"ATriggerSphere", radius:350, existsAfter:true}` (create is correct). Then `volume.set_volume_extent {volumeName:"ReplayCenterTrigger", extent:{500,500,500}}` returned `{actorClass:"TriggerSphere", newExtent:{500,500,500}, existsAfter:true}` — but an immediate `volume.get_volumes_info {filter:"ReplayCenterTrigger"}` read back `extent {1750,1750,1750}`. Mechanism: non-brush volumes take the `SetActorScale3D(NewExtent/100)` fallback (:1318), so `extent {500,500,500}` is applied as actor scale `{5,5,5}` (NOT an absolute world extent), and the 350-radius sphere's bounding-box half-extent becomes 350×5 = 1750; meanwhile the `newExtent` echo (:1325-1329) blindly re-emits the raw request `{500,500,500}`. So `set_volume_extent`'s `extent` param means *absolute half-extent* for brush volumes but *scale×100* for non-brush trigger spheres/capsules — and either way `newExtent` never equals the `get_volumes_info` readback, defeating round-trip verification (the attempt agent's self-report: "newExtent {500,500,500} but readback {1750,1750,1750} … the units do not round-trip"). Same culprit (`volume.set_volume_extent`), same fix requirement as #5: make `newExtent` reflect the post-apply `GetActorBounds()` half-extent (not the raw request) — and the fix must cover the non-brush `SetActorScale3D` branch too, not just the brush rebuild. No new bug; widens confirmed echo-mismatch scope to the non-brush sphere/capsule branch.
- `#3-fix-brush-model-init` `IN-REVIEW` developer — Root-cause fix applied. The three brush helpers in `VolumeHandler.cpp` (`CreateBoxBrushForVolume`/`CreateSphereBrushForVolume`/`CreateCapsuleBrushForVolume`) now route through a new `VolumeHelpers::BuildBoxBrushGeometry` that initializes the volume's `Brush` UModel before running the cube builder — mirroring the engine's own `UActorFactory::CreateBrushForVolumeActor`: `PreEditChange`, `Brush = NewObject<UModel>` + `Brush->Initialize(nullptr,true)` + `Brush->Polys = NewObject<UPolys>`, `GetBrushComponent()->Brush = Brush`, `BrushBuilder = Duplicate(...)`, `CubeBuilder->Build(...)`, then `FBSPOps::csgPrepMovingBrush(Volume)` (builds simple-brush collision + refreshes nav) and `PostEditChange`. Previously the cube builder hit a null `Brush` and `UEditorBrushBuilder::EndBrush` early-returned success without writing any geometry (EditorBrushBuilder.cpp:79-83), yielding the collisionless phantom. This single helper backs the create verb (`SpawnVolumeActor`→`CreateBoxBrushForVolume`, :273) and both setters (`set_volume_extent` :1305, `set_volume_bounds` :1440), so all three affected RPCs are fixed at once — including the TriggerBox scope from #2. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Volume/VolumeHandler.cpp` (helpers + `#include "Model.h"`/`"BSPOps.h"`), `Source/EditorAutomationRpcGateway/EditorAutomationRpcGateway.Build.cs` (added `BSPUtils` private dep for `FBSPOps`). Regression test: `Tests/World/TestVolumeHandlers.cpp` → `FVolumeCreateBlockingVolumeBrushGeometryTest` (`volume.create_blocking_volume.BuildsBrushGeometryWithCollision`) invokes the real handler against the editor world with `extent {300,400,500}`, then asserts `Volume->Brush != null`, `BrushComponent->Brush` wired, `Brush->Polys->Element.Num() > 0`, and `GetActorBounds` half-extent ≈ `{300,400,500}` (±1) — all of which fail if the brush-model init is reverted. Not compiled/tested here; later phase verifies green.
