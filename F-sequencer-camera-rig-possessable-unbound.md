---
id: F-sequencer-camera-rig-possessable-unbound
title: "sequencer.add_camera_rig_rail / add_camera_rig_crane actorPath branch creates an OBJECT-UNBOUND possessable — AddPossessable is called but BindPossessableObject never is, so playback/editor rebinding drives nothing"
status: IN-REVIEW
severity: Medium
category: bug
tags: [sequencer, add_camera_rig_rail, add_camera_rig_crane, possessable, object-binding, bind-possessable-object, sequencer-possessable-unbound, silent-failure, false-success]
encounters: 1
---

# `sequencer.add_camera_rig_rail` / `add_camera_rig_crane` (actorPath branch) bind no object to the possessable they create

Same root-cause family as `B-sequencer-add-actor-unbound-possessable` (add_actor / add_actors / add_camera), split out because it lives in a different verb family with a different actor-resolution path and was not covered by B's repro or its adopted red test.

`AddCameraRigTrackInternal` (`Handlers/Sequencer/SequencerHandler.cpp`, shared by `sequencer.add_camera_rig_rail` and `sequencer.add_camera_rig_crane`) has two binding branches:

- **`actorPath` branch (`:390`)** — `RigActor = LoadObject<AActor>(nullptr, *ActorPath)` then `BindingGuid = MovieScene->AddPossessable(RigActor->GetActorLabel(), RigActor->GetClass())`. It **never** calls `ULevelSequence::BindPossessableObject(BindingGuid, *RigActor, World)`, so no `FLevelSequenceBindingReference` is written. The possessable is object-UNBOUND: the transform + float tracks it then adds (`:414`, `:428`) are keyed to a GUID that resolves to no object, so sequencer playback/evaluation drives nothing and the editor shows an unbound (needs-rebinding) track — the identical defect B fixes for the add-actor family.
- **`else` (spawnable) branch (`:402`)** — `MovieScene->AddSpawnable(ClassName, *DefaultObject)`. This is NOT affected: a spawnable carries its own object template and needs no binding reference (same reasoning as `add_spawnable_from_class`).

## Fix

In the `actorPath` branch only, after `AddPossessable` (`:390`), bind the possessable to the loaded rig actor with the editor world the resolver scans:
```cpp
LevelSequence->Modify();
LevelSequence->BindPossessableObject(BindingGuid, *RigActor, GEditor->GetEditorWorldContext().World());
```
Mirror B's fix exactly (the proven pattern from `TestSequencerControlRigTrack.cpp:155`). Do NOT touch the spawnable branch.

## Acceptance

A regression test that possesses an existing rig actor via the `actorPath` branch and asserts `ULevelSequence::FindBindingFromObject(RigActor, World)` resolves back to the returned `bindingGuid` (the reverse of `BindPossessableObject`) — fails pre-fix (no binding reference), passes post-fix. NOTE the actorPath→world assumption: the test must confirm the loaded rig actor lives in `GEditor->GetEditorWorldContext().World()` so the bind context matches the resolver's scan world (this is the one thing that differs from B's `FindActorByName`/spawn-in-active-world path and is why this is a separate ticket).

## History
- `#2-go` `IN-REVIEW` developer — GO, fix implemented and its regression test verified green. Root cause confirmed: `AddCameraRigTrackInternal`'s actorPath branch (`SequencerHandler.cpp:408`) called `MovieScene->AddPossessable` and never `BindPossessableObject` → object-UNBOUND possessable returned as `success:true`+`mode:"possessed"` (silent false-success). Distinct from `B-sequencer-add-actor-unbound-possessable` (B landed in a DIFFERENT file, `SequenceHandler.cpp`; this one had zero binds) — not a duplicate/regression. SHIPPED in `Handlers/Sequencer/SequencerHandler.cpp`: added `#include "Editor.h"` and, in the actorPath branch only, after `AddPossessable`, `if (BindingGuid.IsValid()) { LevelSequence->Modify(); LevelSequence->BindPossessableObject(BindingGuid, *RigActor, GEditor->GetEditorWorldContext().World()); }` — mirrors B's shipped pattern, covers both `add_camera_rig_rail` and `add_camera_rig_crane` via the shared internal, spawnable branch untouched. Adopted the red test `Tests/Sequencer/TestSequencerCameraRigBinding.cpp` (`PinWright.Sequencer.CameraRigRail.BindsPossessableToObject`): failed pre-fix on the `FindBindingFromObject` binding assertions, compiles clean and passes post-fix (red-green). Left the missing-`FScopedTransaction` hygiene to the separate OPEN ticket `E-sequencer-add-camera-track-no-transaction`.
- `#1-split-from-B` `OPEN` developer — Discovered while fixing `B-sequencer-add-actor-unbound-possessable`: the same AddPossessable-without-BindPossessableObject defect exists at `SequencerHandler.cpp:390` in `AddCameraRigTrackInternal`'s actorPath branch (shared by `add_camera_rig_rail`/`add_camera_rig_crane`). Split off rather than folded into B because it is a distinct verb family, uses a different actor-resolution path (`LoadObject<AActor>` from actorPath vs B's `FindActorByName`/spawn), and is not covered by B's adopted red test. The parent feature `F-sequencer-camera-rig-rail-crane` is DONE and only scoped *adding* the handlers, not object-binding — so this is a fresh defect ticket, not a reopen. `E-sequencer-add-camera-no-actor-path` (IN-REVIEW) is orthogonal (add_camera return shape). Fix = mirror B's one-line `BindPossessableObject` bind in the actorPath branch; leave the spawnable branch untouched.
