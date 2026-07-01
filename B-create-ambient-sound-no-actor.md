---
id: B-create-ambient-sound-no-actor
title: "audio.create_ambient_sound reports success but spawns a bare UAudioComponent on the world, not an AAmbientSound actor"
status: IN-REVIEW
severity: High
category: bug
tags: [audio, ambient-sound, silent-wrong-effect]
---

# audio.create_ambient_sound reports success but spawns a bare UAudioComponent on the world, not an AAmbientSound actor

The `audio.create_ambient_sound` wiki page promises:

> "Spawn an AAmbientSound actor (looping/persistent audio source) at a world
> position with the given Sound asset. Unlike audio.play_sound_at_location
> which auto-destroys, this actor stays in the level until deleted."

In practice the call returns a success-shaped JSON object (`existsAfter:true`)
but does **not** create an `AAmbientSound` actor. Instead it creates a bare
`UAudioComponent` and parents it to an existing world object — the result
reports `componentClass:"AudioComponent"` and `ownerActorPath` pointing at the
map/world itself (`/Game/Maps/ExampleProjectWelcome`), with no new actor. A
subsequent `actor.find_by_class` for `AmbientSound` returns `count:0`,
confirming no actor exists.

This is a silent success-with-wrong-effect: the caller is told the ambient
source exists, but the documented persistent `AAmbientSound` actor it expected
to place/select/delete in the level is never created. The returned
`componentName` (e.g. `AudioComponent_5`) is also not resolvable by
`system.inspect.inspect_object` (`OBJECT_NOT_FOUND`) nor reachable via
`actor.get_components`, so the produced object is effectively undiscoverable
through the normal actor APIs.

**Impact:** Authoring a persistent ambient bed in a level via this RPC is
impossible — the tool reports it worked while leaving no AAmbientSound actor,
and the thing it did create can't be inspected, moved, or deleted through the
actor namespace.

**Workaround:** None via this RPC. (No sibling RPC was found that spawns an
AAmbientSound actor.)

**Fix:** Either actually spawn an `AAmbientSound` actor (matching the doc) and
return its `actorPath`/`actorName`, or correct the doc to describe the
component-on-world behavior and return a resolvable `componentPath`.

## Verbatim repro

`audio.create_ambient_sound`
args:
```json
{"soundPath":"/Game/ExampleContent/Niagara/Audio/NiagaraPopLooping.NiagaraPopLooping","location":[200,100,120],"volume":0.4}
```
result (success-shaped, but a component on the world — no actor):
```json
{"componentName":"AudioComponent_5","assetPath":"/Game/ExampleContent/Niagara/Audio/NiagaraPopLooping","assetName":"NiagaraPopLooping","existsAfter":true,"assetClass":"SoundWave","componentClass":"AudioComponent","ownerActorPath":"/Game/Maps/ExampleProjectWelcome"}
```

confirmation no actor exists — `actor.find_by_class`
args:
```json
{"className":"/Script/Engine.AmbientSound"}
```
result:
```json
{"actors":[],"count":0,"world":"auto","worldPath":"/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome"}
```

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `audio.create_ambient_sound` (NiagaraPopLooping at [200,100,120] vol 0.4): returned `existsAfter:true` with `componentClass:"AudioComponent"` / `ownerActorPath:"/Game/Maps/ExampleProjectWelcome"` and a `componentName` (AudioComponent_5) but no actor. `actor.find_by_class` for `/Script/Engine.AmbientSound` returns `count:0`, contradicting the doc's "Spawn an AAmbientSound actor ... stays in the level until deleted". Silent success with wrong effect.
- `#2-spawn-real-ambient-actor` `IN-REVIEW` developer — Fixed the root cause in `Handlers/Audio/AudioHandler.cpp` (`audio.create_ambient_sound`): replaced the `UGameplayStatics::SpawnSoundAtLocation` call (which only created a transient `UAudioComponent` parented to the world) with a real `World->SpawnActor` of an `AAmbientSound` actor, matching the doc. `AAmbientSound` is a MinimalAPI engine UCLASS so its `StaticClass()` is not exported cross-module — resolved the class via `FindObject<UClass>(nullptr, "/Script/Engine.AmbientSound")` and spawned via the `AActor` base. The actor's `UAudioComponent` is configured with the requested `Sound`/`VolumeMultiplier`/`PitchMultiplier`/`AttenuationSettings`/`ConcurrencySet` and started via `Play(StartTime)`. Response now returns `actorPath`/`actorName`/`actorClass:"AmbientSound"` (via `AddActorVerification`) alongside the asset fields, so the placed ambient bed is selectable/movable/deletable through the actor namespace. Added regression test `FAudioCreateAmbientSoundSpawnsActorTest` ("EditorAutomationRpcGateway.audio.create_ambient_sound.SpawnsAmbientSoundActor") in `Private/Tests/Media/TestAudioHandlers.cpp`: it creates a registered transient `USoundWave`, drives the handler against the live editor world inside `FScopedEditorWorldActorGuard`, and asserts `actorClass=="AmbientSound"`, that an `AAmbientSound` actor (matched by reflection `IsA`) now exists in the world, and that its `UAudioComponent` carries the requested sound — this fails if the fix is reverted to the `SpawnSoundAtLocation` behavior. Did not compile/run (later phase).
