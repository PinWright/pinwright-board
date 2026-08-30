---
id: E-sequencer-add-camera-no-actor-path
title: "sequencer.add_camera returns only actorLabel (non-unique), not the spawned camera's actor path that add_camera_track requires"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [sequencer, add-camera, add-camera-track, actor-path, round-trip, docs, discovery]
---

# `sequencer.add_camera` withholds the spawned camera's actor path that its natural successor `add_camera_track` requires

The obvious two-step "add a camera then cut to it" workflow does not chain.
`sequencer.add_camera` spawns an `ACameraActor`, binds it to the sequence, and
returns `{ bindingGuid, success, actorLabel }`. Its natural successor
`sequencer.add_camera_track` requires `cameraActorPath` — a real actor object
path that the handler resolves with `LoadObject<ACameraActor>(...)`
(`SequencerHandler.cpp:283`). `add_camera` never returns that path, so the
producer's output does not satisfy the consumer's required input.

The label it *does* return cannot be used as the path:

- `sequencer.add_camera_track { cameraActorPath: "SequenceCamera" }` (passing the
  returned `actorLabel`) fails with `[CAMERA_LOAD_FAILED] Failed to load camera
  actor`.

So the caller must drop into a different namespace to recover the path
(`actor.find_by_name { name: "SequenceCamera" }`), and that lookup is **keyed on
a non-unique label**: `add_camera` always spawns the actor with the literal label
`"SequenceCamera"` (`SequenceHandler.cpp:620`), so a sequence (or session) with
more than one added camera returns multiple ambiguous hits. Observed on replay
after two `add_camera` calls:

```
actor.find_by_name { name: "SequenceCamera" } ->
{"count":2,"actors":[
  {"label":"SequenceCamera","name":"CameraActor_7","path":"/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.CameraActor_7", ...},
  {"label":"SequenceCamera","name":"CameraActor_8","path":"/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.CameraActor_8", ...}]}
```

The caller has no in-band way to tell *which* of the two `add_camera` just
created — `add_camera` returned only the binding GUID (a MovieScene binding, not
an actor path) and the shared label. With the right path the chain works:

```
sequencer.add_camera_track {
  sequencePath: "/Game/Cinematics/EstablishingShotReplay",
  cameraActorPath: "/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.CameraActor_7",
  startTime: 0, endTime: 5 }
-> { success: true, cameraActorPath: ".../CameraActor_7", startTime: 0, endTime: 5 }
```

This is an ergonomic gap, not a tool bug — both calls behave correctly; the
friction is the mandatory cross-namespace `actor.find_by_name` detour to recover
a path the spawn handler already had in hand, compounded by the lookup being
ambiguous because the spawned label is a fixed non-unique constant. The wiki
overlay (`docs/wiki-src/sequencer.md`) documents the `add_animation_track`
"capture the returned bindingGuid" handoff but says nothing about the
`add_camera` -> `add_camera_track` handoff, names no field linking them, and
gives no workaround.

**Workaround (today):** after `add_camera`, call
`actor.find_by_name { name: "SequenceCamera" }` and take the returned `path` as
`cameraActorPath` for `add_camera_track` — but disambiguate by hand when more
than one `SequenceCamera` exists (e.g. prefer the highest `CameraActor_N`, or
delete prior ones first).

**Fix (ergonomic, additive):** the `add_camera` handler holds the spawned
`AActor* Spawned` at response time (`SequenceHandler.cpp:618-639`); have it also
emit the actor object path (`Spawned->GetPathName()`, e.g. as `cameraActorPath`
and/or `actorName`) alongside `bindingGuid` and `actorLabel`, so the response is
directly chainable into `add_camera_track`'s `cameraActorPath` with no
`actor.find_by_name` hop and no ambiguity. The field is cheap (the pointer is
already in scope) and mirrors how `add_camera_rig_rail` / `add_camera_rig_crane`
already return `actorPath` (`SequencerHandler.cpp:447`). Cross-link the
`docs/wiki-src/sequencer.md` overlay with a short "add a camera then cut to it"
workflow note (capture `add_camera`'s new path field -> pass as
`add_camera_track.cameraActorPath`). Same shape as
`E-spawn-returns-actor-not-component-path` (a producer verb returning the wrong
path for its consumer), but a distinct namespace/verb/field.

## History
- `#1-initial-repro` `OPEN` reporter — Struggle-audit of an establishing-shot cinematic task (seed `sequencer.add_actors`, which itself round-tripped clean: 5 actors bound + 5 transform tracks verified). The friction is in the neighbor `sequencer.add_camera`: it returns `{bindingGuid, success, actorLabel:"SequenceCamera"}` with no camera actor path, but `sequencer.add_camera_track` requires `cameraActorPath` resolved via `LoadObject<ACameraActor>` (`SequencerHandler.cpp:283`). Replay-confirmed: `add_camera_track {cameraActorPath:"SequenceCamera"}` (the returned label) -> `[CAMERA_LOAD_FAILED] Failed to load camera actor`; `actor.find_by_name {name:"SequenceCamera"}` recovers the path but returns 2 ambiguous hits (CameraActor_7 + CameraActor_8) because `add_camera` always spawns with the fixed label `"SequenceCamera"` (`SequenceHandler.cpp:620`); `add_camera_track` with the real `.../CameraActor_7` path succeeds. Root: producer (`add_camera`) withholds the actor path its natural consumer (`add_camera_track`) requires, forcing a cross-namespace `actor.find_by_name` detour keyed on a non-unique label. Proposed: emit `cameraActorPath`/`actorName` (`Spawned->GetPathName()`) from `add_camera` (`SequenceHandler.cpp:618-639`) and add a workflow note to `docs/wiki-src/sequencer.md`. Dedup: ripgrep across OPEN/DONE/WONTFIX — `E-spawn-returns-actor-not-component-path` (OPEN) is the same shape but `environment.spawn_*`/`componentPath`; `F-rpc-sequencer-get-camera-cut-track` (DONE) + `F-sequencer-camera-rig-rail-crane` are unrelated; no existing ticket about `add_camera`'s return shape.
- `#2-cross-task-repro` `OPEN` reporter — Cross-task aggregation: the SEED-mode `IntroFlythrough` cinematic task (`sequencer.create` focus, 17 calls) hit the identical friction independently. After `sequencer.add_camera {path:/Game/Cinematics/IntroFlythrough}` returned only a binding GUID + the fixed `SequenceCamera` label, the run fell to a cross-namespace `actor.list {filter:"SequenceCamera"}` to recover the spawned camera's actor path (resolved to `CameraActor_7`) before `sequencer.add_camera_track {cameraActorPath:CameraActor_7, ...}` would chain. Verbatim friction note: "sequencer.add_camera_track needs a cameraActorPath but add_camera only returns a binding GUID/label, so I had to actor.list to resolve the spawned camera's actor path - a discoverability gap." Same root as `#1`; this instance used `actor.list` rather than `actor.find_by_name` for the detour, confirming the workaround isn't a single-call-path quirk. Two independent cinematic tasks now pay the same cross-namespace hop — strengthens fix priority.
- `#3-fix` `IN-REVIEW` developer — Implemented the additive producer-side fix. `sequencer.add_camera` (`Handlers/Sequencer/SequenceHandler.cpp`, success branch ~:647-651) now emits `cameraActorPath` (`Spawned->GetPathName()` — the loadable camera object path) and `actorName` (`Spawned->GetName()` — the unique engine-assigned name) alongside the existing `bindingGuid`/`success`/`actorLabel`. The pointer was already in scope, so this is one cheap additive field pair; it mirrors how `add_camera_rig_rail`/`add_camera_rig_crane` already return `actorPath` (`SequencerHandler.cpp:447`). The response is now directly chainable into `add_camera_track {cameraActorPath:...}` with no `actor.find_by_name`/`actor.list` detour and no ambiguity. Regression test added: `FSequenceAddCameraReturnsLoadableActorPathTest` (`EditorAutomationRpcGateway.sequencer.add_camera.ReturnsLoadableActorPath`) in `Source/EditorAutomationRpcGateway/Private/Tests/Media/TestSequencerHandlers.cpp` — creates a registered `/Game` LevelSequence fixture, invokes `add_camera`, asserts `cameraActorPath` is present/non-empty and differs from the (non-unique) `actorLabel`, asserts `actorName` is present, and proves the path is consumable by resolving it the same way the consumer does (`LoadObject<ACameraActor>(nullptr, *CameraActorPath)` succeeds) — fails if the path field is reverted or stops resolving. Wiki: added a `### sequencer.add_camera` H3 to `Docs/wiki-src/sequencer.md` documenting the "add a camera then cut to it" handoff (capture `cameraActorPath` -> pass to `add_camera_track`; never pass the non-loadable `actorLabel`). Not compiled/tested here (later phase).
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
