---
id: B-sequencer-add-actor-unbound-possessable
title: "sequencer possessable-binding verbs (add_actor / add_actors / add_camera) create an OBJECT-UNBOUND possessable — AddPossessable is called but BindPossessableObject never is, so the verb returns success:true + bindingGuid yet the binding resolves to no object (FK Control Rig fails BINDING_NOT_SKELETAL; playback/editor rebinding broken)"
status: OPEN
severity: High
category: bug
tags: [sequencer, add_actor, add_actors, add_camera, possessable, object-binding, bind-possessable-object, sequencer-possessable-unbound, silent-failure, false-success, controlrig, binding-not-skeletal]
encounters: 1
lastSeen: 2026-07-11T08:02:05.5151743+03:00
---

# `sequencer.add_actor` (and `add_actors`/`add_camera`) create a possessable that is never bound to any object — success:true with a bindingGuid, but the binding resolves to nothing

The sequencer possessable-binding verbs resolve the target actor, then call
`UMovieScene::AddPossessable(Label, Class)` to create the possessable entry
and return `success:true` + a `bindingGuid`. They **never** call
`ULevelSequence::BindPossessableObject(Guid, Object, Context)`, so no
`FLevelSequenceBindingReference` is ever written for that GUID. The result
is an **object-unbound possessable**: a track slot with a binding GUID that
resolves to no object. `get_bindings` lists it (name = the actor label),
so nothing in the RPC responses hints the binding is dead — but any
consumer that reverse-resolves the bound object fails:

- `sequencer.add_controlrig_track` (FK) errors `[BINDING_NOT_SKELETAL]` even
  though the skeletal-mesh actor is confirmed present in the editor world,
  because its resolver does the reverse lookup (`FindBindingFromObject`) and
  finds no binding reference to match.
- Sequencer playback/evaluation would drive nothing (the possessable binds
  to no object), and opening the sequence in the editor shows an unbound
  (needs-rebinding) track.

This is a silent false-success: the documented `add_actor -> add_controlrig_track`
FK workflow (and the general "add actor to a sequence" expectation of a
*bound* actor) cannot work, because `add_actor`'s success is a lie about the
binding being functional.

## Root cause (verified in source — ground truth)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp` — every possessable-creating verb calls `AddPossessable` and none calls `BindPossessableObject` (which appears nowhere in the file):

`sequencer.add_actor` (`:782-788`):
```cpp
FGuid BindingGuid = MovieScene->AddPossessable(
    Found->GetActorLabel(), Found->GetClass());
if (MovieScene->FindPossessable(BindingGuid))
{
    Item->SetBoolField(TEXT("success"), true);
    Item->SetStringField(TEXT("bindingGuid"), BindingGuid.ToString());
    MovieScene->Modify();
}
```
`sequencer.add_actors` (`:893-894`) and `sequencer.add_camera` (`:661`) use the identical `AddPossessable(...)`-only pattern. `FindPossessable(BindingGuid)` succeeds (the possessable struct exists in the MovieScene) so the verb reports success — but the possessable is never tied to the world actor, so `LocateBoundObjects`/`FindBindingFromObject` can never resolve it.

The FK resolver is correct and is NOT the culprit — `Handlers/Sequencer/ControlRigSequencerHandler.cpp:104-139` iterates the editor world and asks `Sequence->FindBindingFromObject(Actor, World)` (`:116`, the reverse of `BindPossessableObject`) for each actor; with no binding reference stored it returns an invalid GUID that never equals `BindingGuid`, so `ResolveBoundSkeletalMeshComponent` returns `nullptr` and `add_controlrig_track` sends `BINDING_NOT_SKELETAL` (`:364-370`). `F-sequencer-controlrig-track`'s regression test proves this resolver works once the possessable IS properly bound (its fixture binds a live `SKM_Manny` directly), confirming the defect is upstream in the add-verbs.

## What it should do

After `AddPossessable(...)`, bind the possessable to the resolved actor:
`LevelSeq->BindPossessableObject(BindingGuid, *Found, Found->GetWorld())`
(the reverse of the resolver's `FindBindingFromObject`), so the GUID gets a
`FLevelSequenceBindingReference` and downstream FK Control Rig / evaluation /
editor rebinding all resolve to the actor. Do this on all three possessable
verbs (`add_actor`, `add_actors`, `add_camera`).

## Affected methods (shared root cause — AddPossessable without BindPossessableObject)

- `sequencer.add_actor` — `SequenceHandler.cpp:782-783` (confirmed via replay below)
- `sequencer.add_actors` — `SequenceHandler.cpp:893-894` (same code path, plural)
- `sequencer.add_camera` — `SequenceHandler.cpp:661` (same `AddPossessable`-only pattern; the camera cut track is added but the possessable itself is unbound)

NOT in this family: `sequencer.add_spawnable_from_class` uses a different mechanism (`AddSpawnable`, `:994`) — a spawnable carries its own object template and needs no binding reference, but it is not instantiated in the editor world, so the FK resolver's world-scan still can't find it (a distinct issue, not this AddPossessable defect).

## Verbatim repro (replay-confirmed live at HEAD via mcp__pinwright__call)

1. `sequencer.create {name: CIN_ReplayBind, path: /Game}`
   -> `{"existsAfter":true,"assetClass":"LevelSequence","assetName":"CIN_ReplayBind"}`
2. `actor.spawn {meshPath: /Game/Characters/Mannequins/Meshes/SKM_Manny.SKM_Manny, actorName: ReplayManny}`
   -> `{"actorClass":"SkeletalMeshActor","actorName":"ReplayManny", ... existsAfter:true}` (SkeletalMeshActor present in the editor world)
3. `sequencer.open {path: /Game/CIN_ReplayBind}` -> `{"message":"Sequence opened"}`
4. `sequencer.add_actor {path: /Game/CIN_ReplayBind, actorName: ReplayManny}`
   -> `{"results":[{"name":"ReplayManny","success":true,"bindingGuid":"856AC9EC468C07B91D6502BC1DDE5F93"}]}`
5. `sequencer.get_bindings {path: /Game/CIN_ReplayBind}`
   -> `{"bindings":[{"id":"856AC9EC468C07B91D6502BC1DDE5F93","name":"ReplayManny"}]}`  (binding listed; nothing signals it is object-unbound)
6. `sequencer.add_controlrig_track {sequence: /Game/CIN_ReplayBind, binding: 856AC9EC468C07B91D6502BC1DDE5F93}` (FK)
   -> `[BINDING_NOT_SKELETAL] FK Control Rig requires the binding to resolve to a skeletal-mesh actor in the editor world; none was found. Bind the possessable to a spawned skeletal-mesh actor, or pass an explicit rigClass.`

The SkeletalMeshActor exists in the world (step 2 + `actor.find_by_class SkeletalMeshActor` confirms it), yet step 6 cannot map the step-4 possessable GUID to it because step 4 never wrote a binding reference.

## Distinction from sibling tickets (not dupes — different root cause)

- `E-sequencer-add-remove-actors-name-label-mismatch` (OPEN, ergonomic) — SAME `AddPossessable(GetActorLabel(),...)` line, but a DIFFERENT defect: it is about the add side speaking the internal actor NAME while `get_bindings`/`remove_actors` speak the display LABEL (an identity-domain split on the remove round-trip). This ticket is the orthogonal correctness bug that the possessable is never *object-bound* at all — a distinct root cause (missing `BindPossessableObject`, not the name/label choice). Filing separately per the "different root cause = not a dupe" rule.
- `F-sequencer-controlrig-track` (IN-REVIEW, feature) — the CR-track verbs themselves are implemented and work when the possessable is bound; that ticket's own test binds `SKM_Manny` directly precisely because no RPC produces a bound possessable. This ticket is the upstream binding bug that makes the documented `add_actor -> add_controlrig_track` FK path unreachable end-to-end.

severity rationale: impact=silent-false-success (add_actor returns success:true + a bindingGuid, but the possessable resolves to no object — the caller trusts a lie, and the whole FK-Control-Rig / bound-object workflow silently cannot work) × reach=common (binding an actor into a sequence is a normal, near-every-cinematic-session workflow) -> High.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live at HEAD (SEED-mode `sequencer.key_controls` cinematic-pose task; the seed was never reached — the goal was blocked upstream here). `sequencer.add_actor ReplayManny` returned `success:true` + bindingGuid `856AC9EC…`, `get_bindings` listed the binding, but `add_controlrig_track` (FK) failed `[BINDING_NOT_SKELETAL]` despite the SkeletalMeshActor being confirmed present in the editor world. Root cause verified in source: `SequenceHandler.cpp:782-783` (add_actor), `:893-894` (add_actors), `:661` (add_camera) all call `MovieScene->AddPossessable(GetActorLabel(), GetClass())` and never `ULevelSequence::BindPossessableObject`, so no binding reference exists and the FK resolver's `FindBindingFromObject` reverse lookup (`ControlRigSequencerHandler.cpp:116`) can never match — the resolver itself is correct (F-sequencer-controlrig-track's test proves it works with a properly bound actor). Fix: call `BindPossessableObject(BindingGuid, *Found, World)` after `AddPossessable` on all three verbs. Dedup: ripgrep OPEN+closed (qmd unavailable) — no ticket covers the missing object binding; distinct from `E-sequencer-add-remove-actors-name-label-mismatch` (same line, name/label domain split — different defect) and from `F-sequencer-controlrig-track` (the CR feature, which works once bound). New symptom-family: `sequencer-possessable-unbound`.
