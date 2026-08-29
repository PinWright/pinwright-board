---
id: B-foliage-paint-does-no-ground-projection
title: "foliage.paint runs no trace, no projection and no normal alignment — it writes each instance at the literal XYZ with a zero rotator and unit scale, never reaching FPotentialInstance::PlaceInstance, so a caller who reads the verb's name gets foliage buried in or floating over the ground and a green instancesPlaced count"
status: IN-REVIEW
severity: High
category: bug
tags: [foliage, paint, add_instances, ground-projection, align-to-normal, placement, silent-wrong-output, no-readback, world-origin, silent-drop, misleading-verb-name]
---

# `paint` is `add_instances` with a weaker argument shape and a name that promises otherwise

`foliage.paint` is registered as *"Paint foliage instances at specified locations"*
(`Handlers/Environment/FoliageHandler.cpp:75`). Its entire placement loop is
`Handlers/Environment/FoliageHandler.cpp:221-238`:

```cpp
FFoliageInstance Instance;
Instance.Location = Location;
Instance.Rotation = FRotator::ZeroRotator;
Instance.DrawScale3D = FVector3f(1.0f);
Instance.ZOffset = 0.0f;
```

(`:222-226`, then `Info->AddInstance(FoliageType, Instance, nullptr)` at `:229` / `:233`.)

No line trace. No surface projection. No normal alignment. No Z offset. No random yaw. The
instance goes exactly where the caller's number said, pointing straight up, at scale 1.

## What "paint" means in the engine, and where this verb stops short

The engine's own single-instance placement routine is
`FPotentialInstance::PlaceInstance`
(`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:5506`). Its inputs are a
`HitLocation`, a `HitNormal` and a `HitComponent` — i.e. the result of a trace
(`FPotentialInstance`'s constructor, `:5497`). From those it does everything the word *paint*
implies:

- `Inst.Location = HitLocation;` (`:5523`) — the **hit**, not the requested point
- random pitch and yaw from the type's `RandomPitchAngle` / `RandomYaw` (`:5527-5537`)
- `if (Settings->AlignToNormal) { Inst.AlignToNormal(HitNormal, Settings->AlignMaxAngle); }`
  (`:5546-5549`)
- `ZOffset` applied in the instance's local space, i.e. along the aligned normal (`:5551-5554`)
- a collision check against the world (`AInstancedFoliageActor::CheckCollisionWithWorld`, `:5432`)

`foliage.paint` reaches none of it. It hand-builds an `FFoliageInstance` and calls
`FFoliageInfo::AddInstance` directly, which is the storage call, not the placement call.

**This is also why the foliage type's own settings are inert for anything PinWright places.**
`foliage.add_type` accepts `alignToNormal` (`FoliageHandler.cpp:496`, parsed `:544-545`) and writes
it to `FoliageType->AlignToNormal` (`:618`) — and that property is read in exactly one place,
`PlaceInstance` at `:5546`, which needs a `HitNormal` no PinWright verb produces. A reader skimming
the handler sees an `alignToNormal` parameter in the same file and concludes the gap is covered.
**It is not**: that parameter belongs to a different verb, sets a type-level property, and governs
the interactive Foliage editor's brush, not this verb's writes. Calling this out explicitly because
it is the most likely way this ticket gets wrongly closed.

## Two silent-input defects in the same handler

**Non-object entries in `locations[]` vanish with no record.** The parse loop
(`FoliageHandler.cpp:126-137`) adds a point only when `Val->Type == EJson::Object`; anything else —
a number, a string, an array-form `[x,y,z]` that `foliage.add_instances` accepts — falls through
with no `skipped[]`, no `skippedCount`, and no reason. Its sibling `foliage.add_instances` was fixed
to report exactly this (`NoteSkipped`, `:695`; registration text at `:640` promises it); `paint`
never was.

**An entry with no `x`/`y`/`z` is placed at the world origin.** `double X = 0, Y = 0, Z = 0;`
followed by three `TryGetNumberField` calls that leave the zero in place on a miss
(`:130-134`, and the same shape in the single-`position` branch at `:145-148`). `{}` becomes
`FVector(0,0,0)`. This is the exact behaviour `Docs/wiki-src/level-review.evidence-and-provenance.md:84`
attributes to `actor.spawn_batch` and `foliage.add_instances` — both of which now report `skipped[]`
instead. The doc names the two verbs that were fixed and misses the one that still does it. Filed
separately as `B-evidence-provenance-foliage-skipped-claim-stale`; recorded here because the fix for
`paint` is what makes that doc line correctable rather than merely deletable.

## What the response does and does not say

`:244-250`: `success`, `foliageTypePath`, `instancesPlaced`, `foliageActorPath`, `foliageActorName`,
and `existsAfter: true` — **hardcoded**, not read back. Nothing about where the instances ended up
relative to any surface, and no field a caller could check to discover that no projection happened.
This is the recurring shape: the call succeeds, `instancesPlaced` is correct, and the output is
wrong because the deciding fact — did anything land on the ground — was never reported.

## Side effect worth knowing about

Given a `foliageTypePath` that loads as a `UStaticMesh` rather than a `UFoliageType`, the verb
**creates and saves a new asset**: `/Game/Foliage/Auto_<MeshBaseName>`, a
`UFoliageType_InstancedStaticMesh` with a hardcoded `Density = 100.0f` (`:170-200`, density at
`:193`). The response's `foliageTypePath` then echoes the auto-created path, so a caller who passed
a mesh gets a content-folder asset they did not ask for and did not name. Not this ticket's defect,
but a fixer changing this code path should not be surprised by it.

## Fix

Two coherent options; the second is preferred.

1. **Make `paint` paint.** Add an optional `projectToGround` (default **true**, since the name
   already promises it) that traces down from each supplied XY, places at the hit, and applies the
   foliage type's `AlignToNormal` / `RandomYaw` / `ZOffset` — ideally by routing through
   `FPotentialInstance::PlaceInstance` rather than re-implementing it, so the type's settings and
   the engine's collision check apply for free. Report per-instance `projected: true/false` and the
   hit component, so an unprojected instance is visible in the response. Reuse the `surface` filter
   vocabulary the `spatial` verbs already speak (see `E-ground-preset-excludes-only-foliage-actors`
   for why the preset matters here) rather than inventing a second one.
2. **Or retire the name.** If projection is not going to be implemented, `paint` is a strictly
   weaker duplicate of `foliage.add_instances` — same destination, fewer accepted forms, no
   `skipped[]`, no per-instance rotation or scale — and the honest move is to deprecate it, point
   its doc at `add_instances`, and stop advertising a behaviour the code does not have. A verb that
   cannot do what its name says is worse than a missing verb, because a missing verb sends the
   caller looking.

Either way, fix the two input defects: report `skipped[]` on a non-object entry, and refuse an entry
with no resolvable location instead of silently placing it at the world origin.

## Adjacent, not folded in

`FoliageHandler.cpp` contains **zero** `FScopedTransaction` occurrences — every mutating foliage
verb (`paint`, `remove`, `add_instances`, `create_procedural`, `add_type`) calls `IFA->Modify()`
(`:240`, `:343`, `:349`, `:898`) with no transaction, against `agent-conventions.md`'s *"Wrap EVERY
mutation in `FScopedTransaction`"*. `B-no-undo-redo` (DONE) covered the BPIR/widget verbs only and
does not mention foliage. Not filed here because it is a namespace-wide gap deserving its own
ticket, not a `paint` defect.

## Same shape as

`B-ground-probe-hits-hull-not-render`, `B-niagara-validate-green-while-component-inactive`,
`B-mrq-render-result-omits-bitrate-and-size` — the call succeeds, every number it reports is
correct, and the output is wrong because the deciding number was never reported.

`F-scatter-layout-verb` is the same underlying capability seen from the other side: placement that
respects the ground. That is a missing *layout* verb and this is a defective *placement* verb, so
they are deliberately split — a fixer could land either without the other — but whoever takes one
should read the other, because a scatter layout that hands its transforms to a `paint` that does not
project produces exactly the floating vegetation both tickets exist to prevent.

## Not RPC-verified

Source-read only; the editor was not running for this pass. No `foliage.paint` call was made and no
placed instance was inspected. An editor test would settle one thing worth knowing before choosing
between the two fixes above: whether `FFoliageInfo::AddInstance` applies *any* of the foliage type's
placement settings on its own path (density culling, `ZOffset`, random yaw). Reading
`PlaceInstance` says it does not — every one of those is applied before `AddInstance` is reached —
but that is an inference from the engine's call order, not an observation.

severity rationale: impact=High — silent wrong output on a normal path: the verb's name, its registered summary and the whole vocabulary of foliage painting promise surface projection, the caller builds a scatter on that promise, and the response hardcodes `existsAfter: true` and publishes nothing that could reveal the instances are unprojected; compounded by two literal silent-input defects in the same handler (non-object entries dropped without `skipped[]`, location-less entries placed at the world origin) × reach=normal — `foliage.paint` is one of six verbs in the namespace and the one a caller reaches for first, which is not a "rare edge path" in the rubric's sense, so the bump-down is declined; a reviewer who reads reach as observed usage (this namespace has never been used on this project — zero `InstancedFoliageActor` across ~8,800 placed instances in three builds) lands on Medium, and that is the specific reading being rejected, because observed usage is an encounters signal and the rubric excludes it as a severity input -> High

## History
- `#1-paint-performs-no-trace` `OPEN` reporter — Source-read only, editor not running; no `foliage.paint` call was made. The verb's whole placement loop is `FoliageHandler.cpp:221-238`: `Instance.Location = Location` (`:223`), `Rotation = FRotator::ZeroRotator` (`:224`), `DrawScale3D = FVector3f(1.0f)` (`:225`), `ZOffset = 0.0f` (`:226`), then `FFoliageInfo::AddInstance` (`:229`, `:233`). No trace, no projection, no normal alignment, no random yaw. The engine's actual placement routine is `FPotentialInstance::PlaceInstance` (`InstancedFoliage.cpp:5506`), driven entirely by a trace hit — `Inst.Location = HitLocation` (`:5523`), random pitch/yaw (`:5527-5537`), `AlignToNormal(HitNormal, ...)` (`:5546-5549`), local-space `ZOffset` (`:5551-5554`), world collision check (`:5432`) — and `foliage.paint` never reaches it. Trap flagged explicitly in the body because it is how this gets wrongly closed: `alignToNormal` DOES appear in the same file (`:496`, `:544-545`, `:618`) but belongs to `foliage.add_type`, sets the type-level `UFoliageType::AlignToNormal`, and that property is read in exactly one place — `PlaceInstance:5546` — so it is inert for anything this plugin places. Two further silent-input defects found in the same handler: non-object `locations[]` entries are dropped with no `skipped[]` (`:126-137`), and an entry with no `x`/`y`/`z` is placed at the world origin because the `double X = 0, Y = 0, Z = 0` initialisers survive a failed `TryGetNumberField` (`:130-134`, `:145-148`) — which is exactly what `level-review.evidence-and-provenance.md:84` wrongly attributes to `add_instances` and `spawn_batch`. Response hardcodes `existsAfter: true` (`:250`) and carries nothing about the placement. Dedup: searched the board for `foliage.paint`, `foliage.add_instances`, projection, `AlignToNormal`, scatter and every `B-foliage-*` / `E-foliage-*` file. `B-foliage-create-procedural-empty-callback-noop` (IN-REVIEW) is a different verb and a different mechanism, and is confirmed fixed in this tree — the reflection path through `UProceduralFoliageEditorLibrary::ResimulateProceduralFoliageComponents` is present at `FoliageHandler.cpp:1176-1194`. `E-foliage-get-instances-drops-scale` (IN-REVIEW), `E-foliage-nested-input-schemas-undocumented` (IN-REVIEW), `E-foliage-remove-silent-edge-inputs` (IN-REVIEW) and `E-foliage-add-type-auto-save-undocumented` (WONTFIX) are readback, docs, `remove`, and `add_type` respectively. Nothing on the board mentions `foliage.paint` beyond its name. Recorded but deliberately NOT filed here: `FoliageHandler.cpp` contains zero `FScopedTransaction` occurrences across all five mutating verbs, which is a namespace-wide gap `B-no-undo-redo` (DONE, BPIR-scoped) does not cover.
- `#2-projection-routed-through-seatinstance` `IN-REVIEW` fixer — Every mechanism claim in the body verified in source before acting; all held. `foliage.paint`'s loop did write `Location`/`ZeroRotator`/`FVector3f(1)`/`ZOffset 0` straight into `FFoliageInfo::AddInstance`, and the engine side is confirmed too: `AddInstanceImpl` (`InstancedFoliage.cpp:2275`) only appends to `Instances`, stamps a base id and inserts into `InstanceHash` — it applies *no* placement, so the ticket's "Not RPC-verified" inference about `AddInstance` is correct by source read. `PlaceInstance` (`:5506`) is trace-driven exactly as described. Two ticket details are off: `TraceGroundBelow` lives in `SpatialTraceUtils`, not `GroundPlacementUtils`, and `AInstancedFoliageActor::SetFoliageInstanceTransform` — the obvious engine route for the write — is **unusable from a plugin module**: the class is `MinimalAPI` and that member carries no `FOLIAGE_API`, so it would not link. FIX (option 1, routed, not re-implemented): supplying `surface` now seats each instance through `GroundPlacement::SeatInstance(..., bApply:false)` — the canonical solve, taken as a DRY RUN — and the write goes through `FFoliageInfo::PreMoveInstances` / `Instances[i].Location` / `PostMoveInstances`, because `SeatInstance`'s own applying path is `UpdateInstanceTransform`, which would move the render instance and leave the foliage record and its location hash pointing at the old Z. No new ground math: the percentile/embed solve is never re-derived. `EmbedFraction` is forced to 0 so foliage rests on the surface rather than bedding into it. The component is fetched INSIDE the loop, not before it: `FFoliageStaticMesh` creates its HISM lazily in the first `AddInstance` (`PreAddInstances` -> `Initialize` -> `CreateNewComponent`), so `GetComponent()` is null right after `AddFoliageType`; the up-front refusal is `IsA<UFoliageType_InstancedStaticMesh>` instead. An instance whose column finds no accepted ground is REMOVED (`RemoveInstances` on the last index, a plain truncation) rather than left floating, and reported in `skipped[]` with the solve's own reason. DEFAULT IS OPT-IN, deliberately: `projectToGround` is true iff `surface` was supplied. Default-ON would have had to invent a preset, which the ground module explicitly forbids, and would have broken `TestFoliagePlacementBehaviour.cpp` (a sibling agent's in-flight `F-foliage-namespace-has-no-behavioural-tests` file, written 08:23 today, which paints without a surface and states this ticket "carries its own regression test for the projection itself"). `projectToGround:true` with no `surface` is `INVALID_SURFACE_SPEC`, never a guess. What closes the silent-wrong-output defect on BOTH branches is that `projected` is now on every response, the unprojected branch adds a `warnings[]` line naming `surface` as the remedy, and the projecting branch adds `projectedCount`, `surfacePreset` and a `placed[]` row per instance carrying the reached XYZ and the signed `deltaZCm`. Both input defects fixed too: a non-object entry and an entry missing any of x/y/z now land in `skipped[]`/`skippedCount` (32-row cap, `skippedTruncated`) instead of vanishing or being placed at the world origin; a coordinate-less single `position` is a hard `INVALID_ARGUMENT`. `groundProvenance` is NOT published here — `GroundRpcProvenanceObject` is file-local to `GroundPlacementHandler.cpp`, which `E-ground-preset-excludes-only-foliage-actors` is being worked in right now, so lifting it was left alone. Files: `Handlers/Environment/FoliageHandler.cpp`, new `Tests/Environment/TestFoliagePaintGroundProjection.cpp` (3 behavioural tests, each red before the fix: `paint.SeatsInstancesOntoTheNamedSurface` drops an instance 800 cm onto a spawned floor and re-reads Z from BOTH `FFoliageInfo::Instances` and the HISM's own instance transform, so a write that moved one and not the other fails; `paint.UnprojectedBatchDisclosesThatItDidNotProject`; `paint.LocationWithoutCoordinatesIsReportedNotPlacedAtTheOrigin`), and a `### foliage.paint` overlay section in `Docs/wiki-src/foliage.md`. Not compiled or run — the wave owner builds. Still untouched and still open elsewhere: the missing `FScopedTransaction` across every mutating foliage verb, and the `/Game/Foliage/Auto_<Mesh>` side-effect asset.
