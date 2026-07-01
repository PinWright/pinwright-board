---
id: B-editor-possess-non-pawn-silent-success
title: "editor.possess returns success:true when the target is not a Pawn (POSSESS no-op)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [editor, pie, possess]
---

# editor.possess returns success:true when the target is not a Pawn (POSSESS no-op)

`editor.possess` resolves the named actor, selects it, runs the `POSSESS`
console command, and then **unconditionally** returns `{"success":true}` —
it never checks whether the player controller's possessed pawn actually
changed, nor whether the target is even an `APawn`. The UE `POSSESS` console
command only possesses the selected actor if it is a Pawn; for a non-Pawn
(e.g. a `SkeletalMeshActor`, a `StaticMeshActor`, a light) it is a silent
no-op. The handler reports success regardless, so an agent that hands player
control to a non-Pawn actor is told it worked when nothing happened.

This is a **silent success-with-no-effect**: the only signal the caller has
is `success:true`, and `editor.status.playerControllerPath` reports the
controller object (which never changes), not which pawn it possesses — so the
caller has no way to learn from the response that possession didn't take.

## Repro (verbatim)

Map `ExampleProjectWelcome` (UE 5.7, ContentExamples). `SK_DinoDragon` is a
`SkeletalMeshActor` (not a Pawn) placed in the level; the level's player
controller `PhysicsDemoPlayerController_C_0` possesses `PlayerCharacter_C_0`.

1. `editor.play {}` → `{"success":true}`; `editor.status` → `inPie:true`,
   `playerControllerPath:".../PhysicsDemoPlayerController_C_0"`.
2. `actor.find_by_class {className:"SkeletalMeshActor", world:"pie"}` →
   `SK_DinoDragon` at `.../PersistentLevel.SkeletalMeshActor_5`.
3. `editor.possess {actorName:"SK_DinoDragon"}` → **`{"success":true}`** (the bug).
4. `system.inspect.inspect_object {objectPath:".../PhysicsDemoPlayerController_C_0"}`
   shows the controller's `Pawn` and `AcknowledgedPawn` are **still**
   `.../PersistentLevel.PlayerCharacter_C_0` — i.e. the POSSESS was a complete
   no-op, the SkeletalMeshActor was never possessed, yet step 3 returned success.

Re-possessing the same non-Pawn a second time likewise returns
`{"success":true}` with no state change.

## What it should do

Verify the result of the POSSESS attempt before reporting success. After
`Exec("POSSESS")`, read the controller's possessed pawn and confirm it is now
the requested actor (or pre-check that the target `IsA(APawn::StaticClass())`).
On a non-Pawn target — or any case where the controller's pawn did not change
to the requested actor — return a typed error (e.g. `NOT_A_PAWN` /
`POSSESS_FAILED`) with the actor's class in the message, instead of
fake-success. Optionally echo the resulting `possessedPawnPath` in the success
payload so the caller can confirm the takeover (this would also close the
status-observability gap where `editor.status` only reports the controller,
not its pawn).

**Source:** `Source/PinWright/Private/Handlers/Editor/PIEHandler.cpp` —
`editor.possess` handler runs `GEditor->Exec(GEditor->PlayWorld, TEXT("POSSESS"))`
and immediately sets `success:true` without inspecting the Exec result or the
controller's resulting `Pawn`.

## History
- `#1-initial-repro` `OPEN` reporter — `editor.possess {actorName:"SK_DinoDragon"}` (a SkeletalMeshActor, not a Pawn) in PIE on `ExampleProjectWelcome` returned `{"success":true}`, but `system.inspect.inspect_object` on the player controller showed `Pawn`/`AcknowledgedPawn` unchanged at `PlayerCharacter_C_0` — the POSSESS console command was a no-op yet the RPC reported success. Replay-confirmed: success on first and second possess, controller pawn never changed. Silent success-with-no-effect. Fix: validate the target is a Pawn / verify the controller's pawn actually changed before returning success, else return a typed `NOT_A_PAWN`/`POSSESS_FAILED` error.
- `#2-fix` `IN-REVIEW` developer — Fixed the silent success in `editor.possess`. `Source/PinWright/Private/Handlers/Editor/PIEHandler.cpp`: added an `IsA(APawn::StaticClass())` pre-check (before the PlayWorld gate, since a non-Pawn can never be possessed) that returns a typed `NOT_A_PAWN` error carrying the actor's class; and after `Exec("POSSESS")` it now walks `PlayWorld->GetPlayerControllerIterator()` and confirms a controller's `GetPawn()` equals the requested Pawn — returning `POSSESS_FAILED` if the takeover did not land, instead of unconditional `{success:true}`. Success now echoes `possessedPawnPath` so the caller can confirm the takeover (closes the status-observability gap where `editor.status` reports only the controller, not its pawn). Updated the handler summary to document NOT_A_PAWN/POSSESS_FAILED/possessedPawnPath. Added `#include "GameFramework/Pawn.h"`. Regression test `PinWright.editor.possess.NonPawnRejected` in `Source/PinWright/Private/Tests/EditorOps/TestEditorHandlers.cpp` spawns a non-Pawn StaticMeshActor into the editor world and asserts `editor.possess` returns the `NOT_A_PAWN` error (not `success:true`); it fails if the Pawn pre-check is reverted.
