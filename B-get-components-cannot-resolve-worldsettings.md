---
id: B-get-components-cannot-resolve-worldsettings
title: "actor.get_components cannot resolve WorldSettings by the exact object path actor.find_by_class just returned — the editor-world candidate set comes from GetAllLevelActors(), which excludes AWorldSettings by construction"
status: OPEN
severity: High
category: bug
tags: [actor, get_components, worldsettings, actor-resolution, GetAllLevelActors, ACTOR_NOT_FOUND, decals, asymmetry, weapons]
encounters: 1
lastSeen: 2026-09-06T00:00:00Z
---

# One namespace hands you a path; the next verb in the same namespace says no such actor

`actor.find_by_class {className: "WorldSettings"}` returns the actor and its full object path.
Feeding that exact string straight back to `actor.get_components` returns `ACTOR_NOT_FOUND`. The
same string resolves without complaint through `system.inspect.inspect_object` and `property.get`.

## What was called — measured

```
actor.get_components {"actorName": "WorldSettings", "componentClass": "DecalComponent"}
  -> [ACTOR_NOT_FOUND] Actor or Blueprint not found

actor.find_by_class {"className": "WorldSettings"}
  -> resolves, count 1, path /Game/FPS/Maps/FPS_Compound.FPS_Compound:PersistentLevel.WorldSettings

actor.get_components {"actorName": "/Game/FPS/Maps/FPS_Compound.FPS_Compound:PersistentLevel.WorldSettings"}
  -> [ACTOR_NOT_FOUND] Actor or Blueprint not found        (seconds after the find_by_class above)

system.inspect.inspect_object  <same path>  -> resolves
property.get                   <same path>  -> resolves
```

`actor.get_components` works on ordinary actor paths in the same world in the same session, so this
is not the verb being broken — it is a candidate-set gap that removes exactly one actor.

## Root cause — confirmed in source, not inferred

`actor.get_components` resolves its target with
`McpActorUtils::FindActorByName(nullptr, TargetName)`
(`Plugins/PinWright/Source/PinWright/Private/Handlers/Actor/ComponentHandler.cpp:503`). The
`nullptr` world is load-bearing: with no explicit world, `ResolveActorFiltered` tries the PIE world
and then builds the **editor**-world candidate set from
`GEditor->GetEditorSubsystem<UEditorActorSubsystem>()->GetAllLevelActors()`
(`Plugins/PinWright/Source/PinWright/Private/Utils/ActorUtils.cpp:209`).

`UEditorActorSubsystem::GetAllLevelActors` filters `AWorldSettings` out by name:

```cpp
!Actor->IsA(AWorldSettings::StaticClass())    // Don't add the WorldSettings actor,
                                              // even though it is technically editable
```

`C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/Subsystems/EditorActorSubsystem.cpp:388`

So WorldSettings never enters the candidate array, and the whole precedence ladder — object path,
internal name, label — runs over a set that cannot contain it. That is why **every** key form fails
identically, including the full object path: the resolver is not rejecting the string, it is
searching a list the actor was removed from.

The path fallback does not rescue it either. The `/`-prefixed branch gates on
`UEditorAssetLibrary::DoesAssetExist(ActorName)` (`ActorUtils.cpp:230`), a registry-only lookup; a
level-actor object path (`...:PersistentLevel.WorldSettings`) is not a registry asset, so the probe
returns false and `LoadAsset` is never reached.

### Why the sibling verbs disagree

- `actor.find_by_class` iterates `TActorIterator<AActor>(World, ClassToFind)` directly
  (`Handlers/Actor/QueryHandler.cpp:467`) — `TActorIterator` has no WorldSettings exclusion, so the
  verb sees it and hands back its path.
- `ResolveActorFiltered`'s own explicit-`World` and PIE branches also use `TActorIterator`
  (`ActorUtils.cpp:176`). **Only** the no-world editor branch goes through `GetAllLevelActors`.
- `system.inspect.inspect_object` / `property.get` resolve a UObject by path and never touch the
  actor resolver at all, which is why they work.

So the surface contains a working code path for this case already; `actor.get_components` cannot
reach it, because the verb declares **no `world` parameter** (`ComponentHandler.cpp:481-488`) and
hardcodes `nullptr`. There is no argument the caller can pass to force the `TActorIterator` branch.

## Why this matters beyond WorldSettings itself

`UGameplayStatics::SpawnDecalAtLocation` parents every decal component to the level's
`AWorldSettings` and spawns **no actor**. So for a project that places decals this way — this one
does, for weapon impacts — `actor.get_components {actorName: "WorldSettings", componentClass:
"DecalComponent"}` is the *only* one-call bulk route to decal transforms. With it dead, reading N
decal rotations costs N `property.get` calls, one per component path, after separately discovering
those paths.

The same is true of any component the engine parks on WorldSettings, which is the standard home for
component-only spawns with no owning actor.

## What is asked for

1. **Make `actor.get_components` accept the path forms the inspect verbs accept.** The minimal
   change is to stop losing the actor: when the name is a full object path, resolve it directly
   (the way `inspect_object` does) instead of matching it against a filtered level-actor list.
2. **Or** give the verb the `world` parameter its own resolver already supports, so a caller can
   select the `TActorIterator` branch that has no exclusion.
3. **Or** drop the `GetAllLevelActors` dependency in `ResolveActorFiltered`'s editor branch in
   favour of `TActorIterator`, matching the PIE and explicit-world branches — one candidate-set
   rule for all three, and the asymmetry with `find_by_class` disappears with it.

Whichever lands, an actor that `actor.find_by_class` returns should be resolvable by the path
`actor.find_by_class` returned. That round-trip is the contract being broken.

## Workaround

Read each component individually: obtain the component object paths some other way (an
`inspect_object` on the WorldSettings path, or the `componentPath` a spawn verb returned if the
component was spawned through MCP) and issue one `property.get` per component. For components the
MCP surface did not spawn — game-code decals, the case here — there is no spawn response to lean
on, so the paths must be reconstructed by index.

## Severity

**High.** Impact class: rejecting valid input — specifically, input the same namespace's sibling verb
emitted seconds earlier, and which two other verbs accept — which the rubric puts in the
High/Medium hard-blocker band. Held at High rather than Medium because the failure is total for a
whole class of component (everything parented to WorldSettings), the error message gives the caller
nothing to correct (`ACTOR_NOT_FOUND` on a path that demonstrably exists), and the verb exposes no
parameter that would let a caller route around it. The per-component `property.get` fallback is a
workaround for reading values, not for the bulk enumeration this verb exists to provide.

## Related

- `E-spawned-audio-component-not-actor-readable` (IN-REVIEW, Low, docs) — **the ticket that first
  measured this behaviour**, on audio components rather than decals: 3 dead
  `actor.get_component_property` + 5 dead `actor.get_components` variants against `WorldSettings_1`
  by label, name and full object path, all `ACTOR_NOT_FOUND`. Its `#2` explicitly **rejected**
  fixing the resolver ("Option #2 … rejected as over-scoped — it would special-case a non-placed
  engine singleton in the load-bearing shared actor-resolution path … for a payoff the
  already-returned `componentPath` delivers for free") and shipped a docs-only steer instead. That
  rationale rests entirely on the spawn response carrying a `componentPath`, and it does not hold
  here: **a decal spawned by `UGameplayStatics::SpawnDecalAtLocation` in game code has no MCP spawn
  response at all**, so there is no returned handle to steer anyone to. Filed as a separate ticket
  rather than appended because that one is a docs ticket about audio verbs, is IN-REVIEW on a
  shipped fix that should not be blocked, and its stated disposition is the thing this evidence
  contests. Note also the source finding above narrows the change: the exclusion lives in the
  engine's `GetAllLevelActors`, not in PinWright's ladder, so switching the editor branch to
  `TActorIterator` is a candidate-set alignment rather than the WorldSettings special-case that was
  rejected.
- `E-actor-verbs-reject-actorpath-slot` (IN-REVIEW) — the *key-name* half of `actor.*` identity
  friction (`actorPath` / `objectPath` aliases). Its fix landed on `actor.get_components` and is
  visible in the source read above (`ActorNameParamUtils::ActorNameParamReq`). This ticket is
  downstream of it: the key is now accepted, and the value still does not resolve.
- `E-actor-name-resolution-label-collision` (IN-REVIEW) — documents that the unique internal object
  name is the collision-safe key. That advice fails for WorldSettings, where no key works.
- `B-actor-find-by-class-short-name-fails` (IN-REVIEW) — the sibling verb, which resolves
  WorldSettings fine and is the source of the path this verb rejects.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 4. `actor.get_components {"actorName":"WorldSettings","componentClass":"DecalComponent"}` returned `[ACTOR_NOT_FOUND] Actor or Blueprint not found`; retried with the full object path `/Game/FPS/Maps/FPS_Compound.FPS_Compound:PersistentLevel.WorldSettings`, obtained from `actor.find_by_class {className:"WorldSettings"}` seconds earlier, and got the identical error, while the same path resolved without complaint through `system.inspect.inspect_object` and `property.get` and `actor.get_components` worked on ordinary actor paths in the same world. Root cause **confirmed in source** for this entry (unlike the audio sibling, which inferred it): `actor.get_components` calls `McpActorUtils::FindActorByName(nullptr, TargetName)` (`Handlers/Actor/ComponentHandler.cpp:503`), and with a null world `ResolveActorFiltered` builds its editor-world candidate set from `UEditorActorSubsystem::GetAllLevelActors()` (`Utils/ActorUtils.cpp:209`), which filters WorldSettings out by name — `!Actor->IsA(AWorldSettings::StaticClass())`, `C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/Subsystems/EditorActorSubsystem.cpp:388`. So the whole precedence ladder (object path, internal name, label) runs over a set the actor was removed from, which is why every key form fails identically and why the full object path fails too; the `/`-prefixed fallback does not rescue it either, because it gates on `UEditorAssetLibrary::DoesAssetExist` (`ActorUtils.cpp:230`), a registry-only probe that returns false for a level-actor object path. The asymmetry is fully explained by the same read: `actor.find_by_class` iterates `TActorIterator<AActor>(World, ClassToFind)` (`Handlers/Actor/QueryHandler.cpp:467`) with no exclusion, `ResolveActorFiltered`'s own explicit-world and PIE branches also use `TActorIterator` (`ActorUtils.cpp:176`), and only the no-world editor branch goes through `GetAllLevelActors` — but `actor.get_components` declares no `world` parameter (`ComponentHandler.cpp:481-488`) and hardcodes `nullptr`, so no argument reaches the working branch. Consequence measured on this project: `UGameplayStatics::SpawnDecalAtLocation` parents every decal component to WorldSettings and spawns no actor, so this verb is the only one-call bulk route to decal transforms, and without it reading N decal rotations costs N `property.get` calls after separately discovering N component paths. Ask: accept the path forms the inspect verbs accept (resolve a full object path directly), or expose the `world` parameter the resolver already supports, or align the editor branch onto `TActorIterator` like its two siblings. Dedup: `E-spawned-audio-component-not-actor-readable` (IN-REVIEW, docs) first measured the same ACTOR_NOT_FOUND behaviour on audio components and explicitly rejected the resolver fix as over-scoped on the grounds that the spawn response already returns a usable `componentPath` — that rationale does not cover a decal spawned by game code, which has no MCP spawn response and therefore no returned handle; a cross-reference entry was appended there rather than reopening it. Also distinct from `E-actor-verbs-reject-actorpath-slot` (key-name aliases, whose fix is visible in this source read and which this ticket is downstream of: the key is accepted, the value still does not resolve) and `E-actor-name-resolution-label-collision` (which recommends the internal object name as collision-safe — advice that fails here, where no key works).
