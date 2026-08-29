---
id: B-foliage-paint-ignores-align-to-normal-and-random-yaw
title: "foliage.paint hardcodes every instance's rotation to a zero rotator, ignoring the AlignToNormal and RandomYaw the UFoliageType it auto-created carries by default — four instances on a 33.9-degree slope all read pitch 0 yaw 0 roll 0, so the verb writes a type whose contract it then refuses to honour"
status: IN-REVIEW
severity: High
category: bug
tags: [foliage, paint, align-to-normal, random-yaw, rotation, silent-wrong-output, foliage-type, vegetation, engine-parity]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The verb creates the foliage type, the type says align and randomise, the verb does neither

Measured on a 33.9-degree slope, surface normal `(0.006, 0.558, 0.830)`. All four instances placed
by `foliage.paint` read back `pitch 0, yaw 0, roll 0` — while the `UFoliageType` **the verb itself
auto-created** carries `AlignToNormal = True` and `RandomYaw = True`. Four vertical, identically
oriented plants standing at a right angle to a hillside.

Both flags arrive from the engine's own constructor. The auto-create path is
`Handlers/Environment/FoliageHandler.cpp:383-393`:

```cpp
385:  UFoliageType_InstancedStaticMesh *AutoFT = NewObject<UFoliageType_InstancedStaticMesh>(
386:      FTPackage, FName(*BaseName), RF_Public | RF_Standalone);
388:      AutoFT->SetStaticMesh(StaticMesh);
389:      AutoFT->Density = 100.0f;
390:      AutoFT->ReapplyDensity = true;
```

`AlignToNormal` and `RandomYaw` are never written there because they do not need to be:
`UFoliageType::UFoliageType`
(`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:580-586`) sets
`AlignToNormal = true` (`:585`) and `RandomYaw = true` (`:586`) on every instance of the class. So
the type the verb saves to `/Game/Foliage/Auto_<MeshBaseName>` (`:372`) is a truthful description of
how the plant should be placed, and the verb that wrote it places it some other way.

## The rotation is a constant, and the projection fix does not touch it

The whole placement loop is `FoliageHandler.cpp:457-539`. The instance is constructed at
`:461-465`:

```cpp
461:    FFoliageInstance Instance;
462:    Instance.Location = Location;
463:    Instance.Rotation = FRotator::ZeroRotator;
464:    Instance.DrawScale3D = FVector3f(1.0f);
465:    Instance.ZOffset = 0.0f;
468:    Info->AddInstance(FoliageType, Instance, /*InBaseComponent*/ nullptr);
```

`FoliageType` is in scope on line 468 and is never consulted for rotation. `grep -n
"AlignToNormal\|RandomYaw" FoliageHandler.cpp` returns four hits — `:87` and `:99` (the
`FFoliageScaleAndAlignInput` struct and its parser), `:129`
(`ApplyFoliageScaleAndAlign` writing `FoliageType->AlignToNormal`) and `:958`
(`FoliageType->RandomYaw = RandomYaw`) — all of them on the `add_type` / `create_procedural`
**type-authoring** paths. **Not one of them is a read**, anywhere in the file. Nothing in
`foliage.paint` ever asks a foliage type how it wants to be oriented.

The projection fix does not change this, and it is worth being exact about why, because the two
look like they should have met. When `surface` is supplied, `GroundPlacement::SeatInstance`
(`:491-492`) is run as a **dry run** and only its translation is taken:

```cpp
505:      Placed = Seat.ProposedTransform->GetTranslation();
511:      Info->PreMoveInstances(MakeArrayView(&InstanceIndex, 1));
512:      Info->Instances[InstanceIndex].Location = Placed;
513:      Info->PostMoveInstances(MakeArrayView(&InstanceIndex, 1), /*bFinished*/ true);
```

`Location` is written; `Rotation` is not. The seat solve measures the ground — it is the one thing
in this handler that knows the surface normal — and the handler discards the orientation half of
its own answer.

## What correct looks like, and where it lives

`FPotentialInstance::PlaceInstance`
(`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:5506`) is the engine's
placement path and reads both flags off the settings object:

```cpp
5525:  if (DesiredInstance.PlacementMode != EFoliagePlacementMode::Procedural)
5528:      Inst.Rotation = FRotator(FMath::FRand() * Settings->RandomPitchAngle, 0.f, 0.f);
5530:      if (Settings->RandomYaw)
5532:          Inst.Rotation.Yaw = FMath::FRand() * 360.f;
5534:      else
5536:          Inst.Flags |= FOLIAGE_NoRandomYaw;
...
5546:  if (Settings->AlignToNormal)
5548:      Inst.AlignToNormal(HitNormal, Settings->AlignMaxAngle);
```

`FFoliageInstance::AlignToNormal` is `.../Foliage/Public/InstancedFoliage.h:109`. Note what the
engine needs to do this: `Settings` (the foliage type) and `HitNormal`. The handler has both — the
type at `:468`, the normal inside the seat result it already computes at `:491-492` — and uses
neither. Note also `RandomPitchAngle` at `:5528`, a third type field this handler does not read.

This is also the engine's own answer to the open question the parent ticket left in its **Not RPC
verified** section — whether `FFoliageInfo::AddInstance` applies any placement settings on its own
path. It does not: every one of them is applied before `AddInstance` is reached, and this
measurement is the observation that inference was missing.

## This is a SPLIT, and the other half is verified fixed

`B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) is about **projection**, and its
projection subject is **verified fixed this session**: with `surface` supplied, on this same
33.9-degree slope, the response reported `projected: true`, correct per-instance `deltaZCm`, and a
`groundProvenance` block. That is `#2-projection-routed-through-seatinstance` and
`#3-provenance-published-per-instance` working exactly as written.

So the rotation defect is a **distinct** defect that the projection fix made visible — the pattern
of one defect uncovering the next. Before projection, every instance sat at the literal requested Z
with a zero rotator, and there was no way to tell an unoriented plant from an unplaced one; both
looked like "the verb did nothing useful". Now the plant is provably seated on measured ground and
provably facing the wrong way, and only the second half is left.

The two halves must both be stated because a fixer who reads only this ticket would otherwise
re-litigate the projection, and a fixer who reads only that one would close it while this is still
broken. See the disposition recorded on that ticket for why it was **not** marked DONE.

## Fix

In the loop at `FoliageHandler.cpp:457-539`, derive the rotation from the foliage type instead of
constructing a constant at `:463`:

- Apply `RandomYaw` / `RandomPitchAngle` unconditionally (they need no surface), mirroring engine
  `:5525-5537` including the `FOLIAGE_NoRandomYaw` flag on the else branch — that flag is what
  stops a later reapply from re-randomising a deliberately fixed yaw, so dropping it is not
  cosmetic.
- Apply `AlignToNormal` on the projected branch, where the normal exists. `SeatInstance` already
  computes it; surface it on `FGroundInstanceSeatResult` if it is not already reachable, then call
  `Inst.AlignToNormal(HitNormal, FoliageType->AlignMaxAngle)` (engine `:5546-5548`).
- Write the rotation **inside** the existing `PreMoveInstances` / `PostMoveInstances` bracket at
  `:511-513`, alongside the `Location` write. That bracket exists precisely because writing
  `Instances[i]` alone leaves the component and the location hash stale; a rotation written outside
  it would have the same problem the comment at `:506-510` describes.

On the **unprojected** branch there is no normal, so `AlignToNormal` cannot be honoured. That
branch must then say so, in the same `warnings[]` line it already uses to report `projected:false`
(`:551`, `:560`+) — "alignToNormal was not applied: no surface was supplied". Silently ignoring a
type flag is what this ticket is about; ignoring it with a reason is a legitimate result.

Reusing the engine's `PlaceInstance` outright is the obvious alternative and is worth an explicit
look before hand-rolling the above — it would also bring `DrawScale3D` (hardcoded to `1.0f` at
`:464`, while the type carries a `ScaleX/Y/Z` range) and `ZOffset` (hardcoded `0.0f` at `:465`)
into line for free. Those two are the same defect on two more fields and are named here rather than
filed separately because they are one code change; if a fixer lands only the rotation, they should
say so and the scale/offset half needs its own ticket.

## Same shape as

`B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) carries the fullest statement of the
class: the call succeeds, every number it reports is correct, and the output is wrong because the
deciding number was never reported. Here the deciding number is the surface normal — measured by
this handler, on this code path, in the same loop iteration, and thrown away.

Nearest members:

- `B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) — **the ticket this splits from.**
  Its title already names "no normal alignment" and "a zero rotator", so this is not new scope; it
  is the half of that title that its two fixes did not reach.
- `B-create-procedural-ignores-scale-and-normal-fields` (OPEN, High) — the same pair of fields on
  the sibling verb, and the two together make a clean statement: **alignment works where it was
  fixed and not where it was never wired.** Its `#3` records `alignToNormal` as verified fixed on
  the procedural path while the *scale* write lands on a property that path never reads. So across
  the two verbs, `alignToNormal` is honoured by `create_procedural` and ignored by `paint`, and
  scale is written-but-unread by `create_procedural` and hardcoded by `paint` — four cells, four
  different failure modes, one namespace.
- `E-foliage-get-instances-drops-scale` (OPEN) — the readback side. It is why this took a slope and
  a deliberate rotation readback to find rather than showing up in a routine verify.

## What was NOT done

- No source was modified.
- Only the **projected** branch was measured (`surface` supplied, 33.9-degree slope, four
  instances). The unprojected branch writes the same constant at `:463` by source read and was not
  called.
- `RandomYaw` was observed as absent (all four yaws exactly 0), not statistically tested. Four
  samples of an unseeded 0-360 uniform all landing on exactly 0.0 is conclusive enough, but the
  claim rests on the source line, not on the sample size.
- `DrawScale3D` (`:464`) and `ZOffset` (`:465`) are hardcoded by the same construction and were
  **not** measured. Named in the fix section, not asserted as observed.
- `RandomPitchAngle` (engine `:5528`) is a third unread type field, identified by source read only.

severity rationale: impact=High — the README's High band verbatim, "silent wrong / stale / hardcoded data on a normal path". `Instance.Rotation = FRotator::ZeroRotator` at `:463` is a hardcoded value on the only path the verb has, the response reports `instancesPlaced` and a per-instance `placed[]` block that carries position and provenance and **no orientation at all**, so nothing in the payload can distinguish an aligned scatter from a vertical one. The output is a hillside of plants growing perpendicular to the ground — visually wrong at a glance to a human, and invisible to the caller that made it. NOT Critical: no crash, and nothing is corrupted — the instances are well-formed and removable, and the foliage type on disk is correct. NOT Medium: Medium is a soft blocker doable via a documented workaround, and this is not a blocker at all — the call succeeds and returns a plausible result, so the caller never learns they need a workaround. (`foliage.add_instances` does accept per-instance rotation and is a real alternative, but only for a caller who already knows this ticket exists AND can source per-instance normals themselves, which the verb does not report.) The rubric's escape hatches are for defects the caller can see. x reach: BOTH modifiers declined, for the same reasons the parent ticket records and which are not restated here — `foliage.paint` is one of six verbs in its namespace and the one a caller reaches for first, which is neither an almost-every-session method nor a rare edge path. One addition specific to this half: the defect only shows on sloped ground, which sounds like a narrowing until you notice that flat ground is where foliage placement is trivial and slopes are the case the verb exists for. High stands unmodified, matching the parent ticket it splits from.

## History
- `#1-rotation-hardcoded-to-zero-rotator` `OPEN` reporter — Measured live against a running editor on a 33.9-degree slope, surface normal `(0.006, 0.558, 0.830)`: all four instances placed by `foliage.paint` read back `pitch 0, yaw 0, roll 0`, while the `UFoliageType` the verb auto-created carries `AlignToNormal = True` and `RandomYaw = True`. All line numbers re-derived this session at HEAD. Mechanism: the placement loop (`FoliageHandler.cpp:457-539`) constructs the instance at `:461-465` with `Instance.Rotation = FRotator::ZeroRotator` (`:463`), `DrawScale3D = FVector3f(1.0f)` (`:464`) and `ZOffset = 0.0f` (`:465`), then calls `Info->AddInstance(FoliageType, Instance, nullptr)` (`:468`) — `FoliageType` is in scope and is never consulted. `grep -n "AlignToNormal\|RandomYaw" FoliageHandler.cpp` gives four hits (`:87`, `:99`, `:129`, `:958`) and **every one is a WRITE on the type-authoring paths**; there is no read anywhere in the file. The two flags come from the engine constructor `UFoliageType::UFoliageType` (`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:580-586`, `AlignToNormal = true` `:585`, `RandomYaw = true` `:586`), which is why the auto-create at `FoliageHandler.cpp:383-393` carries them without writing them — so the verb saves a type to `/Game/Foliage/Auto_<MeshBaseName>` (`:372`) describing placement it then refuses to perform. The projection fix does not reach it: `SeatInstance` (`:491-492`) is a dry run and only `GetTranslation()` is taken (`:505`), written as `Location` inside the `PreMoveInstances`/`PostMoveInstances` bracket (`:511-513`); the handler measures the ground and discards the orientation half of its own answer. Correct behaviour and where it lives: `FPotentialInstance::PlaceInstance` (engine `:5506`) reads `Settings->RandomYaw` (`:5530-5532`, else `FOLIAGE_NoRandomYaw` `:5536`), `Settings->RandomPitchAngle` (`:5528`) and `Settings->AlignToNormal` (`:5546-5548`, calling `FFoliageInstance::AlignToNormal`, `Foliage/Public/InstancedFoliage.h:109`). That also settles the open question in the parent ticket's "Not RPC verified" section: `AddInstance` applies none of these itself — they are all applied before it is reached. **SPLIT, both halves stated:** `B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) is about projection, and its projection subject is VERIFIED FIXED this session — `projected: true`, correct per-instance `deltaZCm` and a `groundProvenance` block on this same slope, i.e. its `#2` and `#3` working as written. The rotation defect is therefore distinct and was made visible by that fix: before it, an unoriented plant and an unplaced one were indistinguishable. It is nonetheless inside that ticket's TITLE ("no normal alignment ... a zero rotator"), which is why no DONE verdict was recorded there — see the disposition appended to it. NOT DONE: no source modified; only the projected branch was called (the unprojected branch writes the same constant by source read); `RandomYaw`'s absence rests on the source line, with four exact-zero yaws as corroboration rather than a statistical test; `DrawScale3D` and `ZOffset` are hardcoded by the same construction and were NOT measured; `RandomPitchAngle` is a third unread type field found by source read only. Dedup: `grep -ril` over the board for `AlignToNormal`/`RandomYaw`/`foliage.paint` returns the parent ticket, `B-create-procedural-ignores-scale-and-normal-fields` (OPEN, High — the same pair of fields on the sibling verb; its `#3` records `alignToNormal` verified fixed there while the scale write lands on a property that path never reads, so across the two verbs alignment works where it was fixed and not where it was never wired) and `E-foliage-get-instances-drops-scale`. None claims `foliage.paint` ignores the type's rotation contract. No umbrella filed; the class statement stays in the parent ticket and is referenced.
- `#2-rotation-read-off-the-type` `IN-REVIEW` developer — `foliage.paint` now derives every instance's rotation from the resolved `UFoliageType` instead of writing `FRotator::ZeroRotator`, mirroring `FPotentialInstance::PlaceInstance`'s non-procedural branch (`InstancedFoliage.cpp:5525-5548`) field for field. In `FoliageHandler.cpp`, four knobs are read once after the type resolves (`RandomYaw`, `RandomPitchAngle`, `AlignToNormal`, `AlignMaxAngle`); the loop then sets `Instance.Rotation = FRotator(FMath::FRand() * RandomPitchAngle, 0, 0)` before `AddInstance` (so it reaches the component through `GetInstanceWorldTransform()` with no second write), sets `Yaw = FMath::FRand() * 360` when `RandomYaw` and stamps `FOLIAGE_NoRandomYaw` when not. `AlignToNormal` is applied on the projected branch INSIDE the existing `PreMoveInstances`/`PostMoveInstances` bracket, alongside the `Location` write, against `Seat.Seat.Contact.AverageNormal` — the normal the seat solve had already measured under this instance's own footprint, so no second trace was added; guarded on `SupportedColumns > 0` and a non-degenerate normal. RESPONSE: `placed[]` now carries `pitch`/`yaw`/`roll` read back off `FFoliageInfo::Instances` after every write, plus `alignedToNormal` off the engine's own `FOLIAGE_AlignToNormal` flag, and is published on BOTH branches (`deltaZCm`/`groundProvenance` stay projected-only); a new `rotation` block echoes what the type asked for next to the MEASURED `alignedCount`; and an `AlignToNormal` that could not be honoured appends a `warnings[]` line naming it, after the existing projection warning rather than displacing it. NO SEED PARAMETER was added — `FMath::FRand` is what engine painting draws from and the ticket ruled an unrequested `seed` out, so the tests assert distribution rather than exact values. REGRESSION TESTS (new file `Private/Tests/Environment/TestFoliagePaintTypeRotation.cpp`, both red against the zero rotator): `PinWright.foliage.paint.AppliesTheTypesAlignToNormalAndRandomYaw` paints 12 instances onto a real 30-degree slope (an engine cube spawned through `actor.spawn` with `rotation.pitch = 30`, scaled 8x on XY) and asserts no instance is left at identity, that each instance's own +Z axis lies within 2 degrees of the slope normal read off the spawned actor, that `FOLIAGE_AlignToNormal` is set, that the yaw diameter across the batch exceeds 45 degrees (wrap-safe via `FindDeltaAngleDegrees`; ~4e-9 flake probability), that the HISM's own instance transform rotated with the record, and that `placed[0]`'s pitch/yaw/roll match the stored instance — then paints a second type with `randomYaw:false` on the same slope as a control, asserting one shared yaw and `FOLIAGE_NoRandomYaw` while still aligned, which is what separates "reads the flag" from "always randomises". `PinWright.foliage.paint.UnhonouredAlignToNormalIsReportedNotDropped` covers the literal branch: yaws still vary, `alignedCount` is 0, no instance claims `FOLIAGE_AlignToNormal`, and a warning names `alignToNormal`. NOT DONE, needs its own ticket as this ticket's Fix section allows: `DrawScale3D` is still hardcoded to `1.0f` and `ZOffset` to `0.0f`, so the type's `ScaleX/Y/Z` and `ZOffset` ranges remain unread — the engine's `PlaceInstance` was NOT reused wholesale (this handler's ordering is AddInstance-then-seat because the HISM is created lazily by the first AddInstance, so the hit normal does not exist when `PlaceInstance` would need it, and `PlaceInstance` also rewrites `Location` and runs a collision check this path does not want). Both the hardcoding and its scope are now stated in the verb's own description and in `Docs/wiki-src/foliage.md`. NOT COMPILED and NOT RUN: per the wave's rules the orchestrator builds and runs the suite; `Content/Python/check_test_ids.py` was run and reports CLEAN (4732 ids, no dot-prefix collisions). `FoliageHandler.cpp` references no `ErrorCodes::` symbol before or after this change, so no error-code adoption was triggered.
- `#3-scale-half-now-has-its-ticket` `IN-REVIEW` reporter — **Status deliberately NOT changed; no code touched.** The scale/offset half this ticket asked for ("say so and the scale/offset half needs its own ticket") is now **`B-foliage-paint-hardcodes-scale-and-zoffset`** (OPEN, Medium). Verified against the current working tree, which carries `#2`'s uncommitted rotation fix: `Instance.DrawScale3D = FVector3f(1.0f); Instance.ZOffset = 0.0f;` are still literals, now at `FoliageHandler.cpp:919-920` (moved from the `:464-465` this ticket cites, by `#2`'s own insertion), under the hand-off comment `#2` wrote at `:916-918` naming this ticket as the owner. Rated **Medium**, not High like the rotation half, on one difference the new ticket argues in full: `#2` also added a disclosure sentence to the verb's registration string (`:548` — "Instance SCALE and ZOffset are still hardcoded to 1 and 0 and do NOT read the type's ScaleX/Y/Z or ZOffset ranges") and names `foliage.add_instances` as the workaround in the same breath, which moves it from the silent-wrong-data class to the documented-workaround class. The escalation condition is recorded there: removing that sentence, or adding `minScale`/`maxScale` params to `paint` that are then ignored, returns it to High. **Scope for this ticket's tester is unchanged and is the rotation half only** — `#2`'s fix does not claim scale, and `DONE` should not be held on it.
