---
id: B-actor-spawn-rootless-actor-ignores-location
title: "actor.spawn of a class with no root component silently ignores location, rotation and scale — every setter on that path is a documented no-op with no root, and the response echoes no transform at all, so the actor sits at the origin and nothing in the reply says so"
status: OPEN
severity: Medium
category: bug
tags: [actor, spawn, spawn_from_blueprint, location, transform, root-component, rootless, silent-noop, no-readback, missing-echo, world-origin, class-dependent]
---

# A declared parameter that is inert for a class of inputs, and a response with nothing to check it against

`actor.spawn` (`Handlers/Actor/SpawnHandler.cpp:57`) declares

```cpp
RPC_PARAM_OPT("location", "object", "World-space location as {x,y,z} in unreal units (cm); defaults to origin."),
```

(`:62`), reads it at `:74` as `FVector Location = Ctx.GetVector(TEXT("location"), FVector::ZeroVector);`,
and places with it twice — once through `SpawnActorInWorld` (`:41-53`, which calls
`World->SpawnActor(ClassToSpawn, &Location, &Rotation, SpawnParams)` at `:52`), and again immediately
after at `:180-181`:

```cpp
Spawned->SetActorTransform(FTransform(Rotation, Location, Scale), false,
                           nullptr, ETeleportType::TeleportPhysics);
```

**Both are no-ops on a class with no root component, and both fail quietly.**

- `AActor::SetActorTransform` (`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/Actor.cpp:5156-5180`)
  is `if (RootComponent) { … return true; } … return false;`. The handler discards the return.
- The spawn-time application is guarded the same way, in the engine's own words:
  *"Set the actor's world transform if it has a native rootcomponent."* —
  `AActor::PostSpawnInitialize`, `Actor.cpp:4304-4306`, where
  `USceneComponent* const SceneRootComponent = FixupNativeActorComponents(this);` gates the
  `SceneRootComponent->SetWorldTransform(FinalRootComponentTransform, …)` at `:4324`. With no root,
  the `UserSpawnTransform` that `UWorld::SpawnActor` carried (`LevelActor.cpp:621`, passed at `:755`)
  is simply dropped.
- The read-back is guarded too: `AActor::GetActorLocation()` is
  `TemplateGetActorLocation(ToRawPtr(RootComponent))`
  (`Runtime/Engine/Classes/GameFramework/Actor.h:2501-2505`), so it reports `(0,0,0)` — not because
  the actor is at the origin but because there is nothing to ask.

`rotation` and `scale` go the same way through the same call. `AActor::SetActorScale3D`
(`Actor.cpp:5078-5084`) is worse still — it returns `void`, so even the discarded `false` does not
exist.

**Measured:** spawning a bare `AActor` with an explicit `location` succeeds, and
`actor.get_transform` on the result reports `(0,0,0)`.

## Which classes reach this

The dead-looking `AActor::StaticClass()` fallback at `:176` is unreachable — `CLASS_NOT_FOUND` at
`:156-160` already rejected everything that failed to resolve, so a typo'd `classPath` does **not**
silently become a bare actor. The rootless path is entered only by asking for a rootless class, which
is a real and reachable set: bare `AActor` itself, every `AInfo` subclass
(`Runtime/Engine/Classes/GameFramework/Info.h:20` — `AGameModeBase`, `AWorldSettings`,
`ALevelScriptActor`, …), and any Blueprint whose SCS establishes no scene root. The registration's own
summary invites it: *"classPath accepts UClass names, /Script paths, BP asset paths"* (`:57`), with no
statement that some of those ignore the placement params.

`actor.spawn_from_blueprint` is the identical shape — param at `:285`, read at `:300`, spawn at
`:346`, `SetActorTransform` at `:348-349` — and a component-less Blueprint is exactly the input that
reaches it.

## Two separable halves — keep them separate

**(a) The silent no-op.** A declared parameter with a documented default is discarded for a class of
inputs, with no error, no warning, and no `warnings[]` entry.

**(b) The missing echo.** `actor.spawn`'s response carries no transform at all. Built at `:240-275`:
an `actor` sub-object with `id` / `name` / `label` / `objectName` / `path` (`:243-253`), `actorPath`
(`:255`), `classPath` (`:257-260`), `meshPath` (`:262-265`),
`SpawnMaterialUtils::AddMaterialReport` (`:267-268`), then `AddActorVerification(Data, Spawned)`
(`:270`) — which writes `actorPath`, `mapPath`, `actorName`, `actorLabel`, `actorObjectName`,
`actorGuid`, a hardcoded `existsAfter: true` and `actorClass`
(`Utils/AssetUtils.cpp:1557-1585`) and nothing else. **Proven by grep:** the only occurrences of
`location`/`Location` in the whole 384-line file are the two param declarations (`:62`, `:285`), the
two reads (`:74`, `:300`), the `SpawnActorInWorld` signature and call (`:42`, `:52`), the four
placement calls (`:177`, `:180`, `:346`, `:348`) and one `UE_LOG` in `spawn_from_blueprint`
(`:335-337`). No response field.

**(b) is what makes (a) undetectable.** With an echoed achieved location the no-op is one comparison
away; without it the only route is a second `actor.get_transform` call that a caller has no reason to
make, because nothing suggested the first call failed. Neither half has a ticket of its own, and (b)
is worth fixing even if (a) is fixed by refusal — an echoed transform is the readback that would
catch the next placement gap on this verb too.

## Contradicts a claim on an IN-REVIEW ticket

`E-spawn-no-scale-param` (IN-REVIEW, Medium) argues, verbatim:

> "`location` and `rotation` are already first-class spawn params; `scale` is the missing third
> component of the same transform, and it is just as commonly non-default at spawn time as the other
> two."

They are first-class in the schema and inert on the same class of inputs `scale` is. Its shipped fix
inherits the defect rather than avoiding it: its `#2` records applying scale *"post-spawn with
`SetActorScale3D(Scale)`"*, and the current source has since folded that into the single
`SetActorTransform(FTransform(Rotation, Location, Scale), …)` at `:180` / `:348` — **both spellings
are no-ops with no root**, so the ergonomic that ticket shipped is silently unavailable on exactly
the classes this ticket is about. That file was not edited.

**Its regression test cannot see it.** `PinWright.actor.spawn.AppliesScale`
(`Tests/World/TestActorHandlers.cpp:97-99`) spawns `/Script/Engine.PointLight` (`:131`) and asserts
`GetActorScale3D()`. `APointLight`'s CDO sets `RootComponent = PointLightComponent`
(`Runtime/Engine/Private/Light.cpp:192`), so the fixture has a root and the assertion exercises the
working branch. The rootless case is untested on both verbs.

## Fix

Three options; **refuse, with a diagnostic** is the recommendation.

1. **Refuse a rootless class.** `CLASS_NOT_FOUND`'s sibling: an error naming the class, naming
   `RootComponent` as the reason, and naming the remedy (`actor.add_component` to attach a scene
   root, or a class that has one) — the plugin's error convention requires actionable context, and
   this is a caller mistake that can be decided before `SpawnActor`, exactly like the material
   pre-flight the handler already runs at `:95-108` for the same stated reason ("failing after
   SpawnActor would strand an orphan actor in the level"). **This is a behaviour change** for any
   caller deliberately spawning a bare actor as a container — a real pattern, and the reason this is
   not obviously the right call. Mitigate by refusing only when a placement param was actually
   supplied: a rootless spawn with no `location`/`rotation`/`scale` is unambiguous and should keep
   working.
2. **Attach a default scene root before placing.** Matches what the editor itself does —
   `UActorFactoryEmptyActor::SpawnActor` creates a `DefaultSceneRoot` and places it
   (`Editor/UnrealEd/Private/Factories/ActorFactory.cpp:1318-1326`) — and makes `location` mean what
   it says for every class. Costs a component the caller did not ask for, which is a quieter
   surprise than the current one but still a surprise, and it changes what the actor serialises as.
3. **Echo the achieved location** and let the caller see the no-op. Necessary regardless (half (b)),
   sufficient for nobody: it turns a silent failure into a visible one without making the parameter
   work.

Recommendation: **1 for half (a), gated on a placement param actually being supplied, plus 3 for half
(b) unconditionally.** Option 2 is rejected as the default because inventing a component to make a
parameter apply is the kind of hidden write this board keeps filing tickets about; it belongs behind
an explicit opt-in if anyone wants it. Whichever lands, the regression test must use a rootless
fixture — bare `AActor` or an `AInfo` subclass — because the existing one structurally cannot fail.

## Same shape as

`B-foliage-paint-does-no-ground-projection` carries the fullest statement of the class: *the call
succeeds, every number it reports is correct, and the output is wrong because the deciding number was
never reported.* Here the deciding number is the achieved location, and the response does not merely
get it wrong — it does not contain it.

`B-spawned-volumes-have-no-brush-geometry` (OPEN, High) — filed this session, and the other
`actor.spawn` class-dependent silent gap: the same verb, the same success envelope, a different thing
that is missing because of what the class is. A fixer should read both before deciding whether
`actor.spawn` needs a general per-class capability pre-flight rather than two point fixes.

`B-create-ambient-sound-no-actor` (IN-REVIEW, High) — root-component-shaped, different verb: it
creates a bare component parented to the world with no actor at all. Related by mechanism, not by
code path.

`B-create-ambient-sound-location-object-dropped` (IN-REVIEW, Medium) — **a triager must not merge
this one.** Same observable (a `location` is supplied, the thing lands at the origin, the response
does not say so) and a completely different cause: there the object-form `location` is never parsed;
here it is parsed correctly, passed correctly, and discarded by the engine for want of a root. A fix
for either leaves the other exactly as broken.

`E-spawn-no-scale-param` (IN-REVIEW, Medium) — the ticket whose premise this contradicts, and whose
shipped fix is inert on the same inputs.

## Verification status

The symptom is measured: a bare `AActor` spawned with an explicit `location` reports `(0,0,0)` from
`get_actor_location()`, and the response carries no location field to contradict it. The mechanism is
source-read — the three engine guards (`Actor.cpp:5156`, `:4304-4306`, `Actor.h:2501-2505`) and the
response's contents were traced, not observed under a debugger. Not measured: whether
`actor.spawn_from_blueprint` on a component-less Blueprint behaves identically. It is the same code
shape at `:346-349` and this ticket asserts it does, but the second verb was not called.

severity rationale: impact=Medium — a declared parameter with a documented default ("defaults to origin", `:62`) is silently discarded for a class of inputs, and the caller has a workaround once they know it exists: spawn a class that has a root, or attach one with `actor.add_component` and place afterwards. Deliberately NOT High: the High band is a caller trusting a result that is a lie, and this response asserts nothing false — `actorPath`, `actorClass` and `existsAfter: true` are all correct, and it never claims a location; it is an omission that permits a silent no-op, not an assertion of wrong data, which is the line separating this from `B-create-procedural-density-writes-paint-density` (filed this session), where the wrong value is positively echoed and reads back correct. NOT Low: the rubric's Low band is friction that costs a `Read` or a lookup, and this costs a prop placed 17 km from where it was asked for with nothing in the reply to reveal it. The missing echo (half b) is the argument for High and is rejected on the same ground — no signal is not the same as a false signal — but it is the reason this is not lower, because without it half (a) is undiscoverable without a source dive. Not Critical: nothing is corrupted, the actor exists and is one `actor.set_transform` away from correct once a root exists × reach modifiers cancel and are BOTH declined explicitly: `actor.spawn` is an every-session verb, which argues the bump up, while the rootless branch is the rare edge of it — every class the registration advertises by name (StaticMeshActor, SkeletalMeshActor, and every mesh/BP path that auto-picks one) has a root — which argues the bump down; the rubric's modifier attaches to the affected path, not to the verb's overall traffic, so the two readings offset and no net modifier applies -> Medium

## History
- `#1-rootless-spawn-drops-placement` `OPEN` reporter — Symptom measured: `actor.spawn` of a bare `AActor` with an explicit `location` succeeds and `get_actor_location()` reports `(0,0,0)`. Mechanism source-read; every citation re-derived against this tree. `actor.spawn` registers at `Handlers/Actor/SpawnHandler.cpp:57`, declares `location` with "defaults to origin" at `:62`, reads it at `:74`, and applies it twice — `World->SpawnActor(ClassToSpawn, &Location, &Rotation, SpawnParams)` via `SpawnActorInWorld` (`:41-53`, call at `:52`, invoked `:177`) and `Spawned->SetActorTransform(FTransform(Rotation, Location, Scale), false, nullptr, ETeleportType::TeleportPhysics)` at `:180-181`. Both are guarded on `RootComponent`: `AActor::SetActorTransform` returns `false` without one (`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/Actor.cpp:5156-5180`; the return is discarded), and the spawn-time application is gated by `PostSpawnInitialize`'s own comment "Set the actor's world transform if it has a native rootcomponent" (`Actor.cpp:4304-4306`, write at `:4324`), so the `UserSpawnTransform` (`LevelActor.cpp:621`, passed `:755`) is dropped. `GetActorLocation()` is `TemplateGetActorLocation(RootComponent)` (`Runtime/Engine/Classes/GameFramework/Actor.h:2501-2505`), so the `(0,0,0)` read-back reports the absence, not a position. `rotation` and `scale` ride the same call; `SetActorScale3D` (`Actor.cpp:5078-5084`) returns `void`, so it has no failure signal at all. NEGATIVE PROVEN for the response: `actor.spawn`'s reply is built at `:240-275` — actor sub-object (`:243-253`), `actorPath` (`:255`), `classPath` (`:257-260`), `meshPath` (`:262-265`), `AddMaterialReport` (`:267-268`), `AddActorVerification` (`:270`) — and `AddActorVerification` (`Utils/AssetUtils.cpp:1557-1585`) writes only `actorPath`/`mapPath`/`actorName`/`actorLabel`/`actorObjectName`/`actorGuid`/hardcoded `existsAfter: true`/`actorClass`. Grepping `location|Location` across all 384 lines returns only the two param decls, two reads, the helper signature/call, four placement calls and one `UE_LOG` in `spawn_from_blueprint` — there is no response field, so nothing to compare against. Two halves kept separate on purpose: (a) the silent no-op, (b) the missing echo, which is what makes (a) undetectable without an unmotivated second `actor.get_transform`; neither has a ticket. Reachability checked rather than assumed: the `AActor::StaticClass()` fallback at `:176` is UNREACHABLE because `CLASS_NOT_FOUND` at `:156-160` already rejected unresolvable input, so a typo'd `classPath` does not become a bare actor; the path is entered only by explicitly asking for a rootless class — bare `AActor`, any `AInfo` subclass (`GameFramework/Info.h:20`), or a Blueprint whose SCS has no scene root, which is also what reaches `actor.spawn_from_blueprint` (same shape, `:285`/`:300`/`:346`/`:348-349`). CONTRADICTS AN IN-REVIEW TICKET, quoted not edited: `E-spawn-no-scale-param` (IN-REVIEW, Medium) argues "`location` and `rotation` are already first-class spawn params; `scale` is the missing third component of the same transform" — they are first-class in the schema and inert on the same inputs. Its `#2` records the fix as `SetActorScale3D(Scale)` post-spawn; current source has folded that into the single `SetActorTransform` at `:180`/`:348`, and both spellings no-op without a root, so the ergonomic it shipped is silently unavailable on these classes. Its regression test cannot see this: `PinWright.actor.spawn.AppliesScale` (`Tests/World/TestActorHandlers.cpp:97-99`) uses `/Script/Engine.PointLight` (`:131`), and `APointLight`'s CDO sets `RootComponent = PointLightComponent` (`Runtime/Engine/Private/Light.cpp:192`) — fixture verified before the claim was made. Fix recommended: refuse a rootless class with a diagnostic naming `RootComponent` and the remedy, gated on a placement param actually having been supplied so a deliberate bare-actor container still spawns; plus echo the achieved transform unconditionally. Attaching a `DefaultSceneRoot` (what `UActorFactoryEmptyActor` does, `Editor/UnrealEd/Private/Factories/ActorFactory.cpp:1318-1326`) is rejected as a default because it is a hidden write. Any regression test must use a rootless fixture; the existing one structurally cannot fail. Dedup: grepped the board for `rootless`, `RootComponent`, `no root component`, `actor.spawn`, "ignores location", "lands at the world origin". `B-create-ambient-sound-location-object-dropped` (IN-REVIEW, Medium) is the nearest match and is explicitly called out in the body as NOT mergeable — same observable, different cause (its `location` is never parsed; this one's is parsed and then discarded by the engine), so fixing either leaves the other broken. `B-create-ambient-sound-no-actor` (IN-REVIEW, High) is root-component-shaped but a different verb creating a bare component under the world. `B-spawned-volumes-have-no-brush-geometry` (OPEN, High, filed this session) is the other class-dependent silent gap on this same verb. The `E-scs-add-component-root-*` tickets are Blueprint SCS authoring, not level spawns. `F-actor-list-omits-location` is bulk projection on `actor.list`, a different ask. Nothing on the board covers either half of this.
