---
id: B-scs-set-property-bodyinstance-collision-silent-noop
title: "blueprint.scs.set_property on BodyInstance.CollisionEnabled / BodyInstance.CollisionProfileName returns success but the spawned component keeps QUERY_AND_PHYSICS — a class-template collision write that never reaches the instance"
status: IN-REVIEW
severity: High
category: bug
tags: [blueprint, scs, set_property, body-instance, collision, silent-false-success, derived-state, nested-struct, dot-path, character, viewmodel]
---

# The SCS template collision write is stored and never applied

## Symptom

`blueprint.scs.set_property` accepts a dot-path into `BodyInstance` and reports success:

```
call("blueprint.scs.set_property", {
  blueprintPath: "/Game/FPS/Player/BP_WeaponStub_PlayerTest",
  componentName: "Body",
  propertyName: "BodyInstance.CollisionEnabled",
  propertyValue: "NoCollision"})
-> {"success": true, "message": "Property 'BodyInstance.CollisionEnabled' set on component 'Body'",
    "source": "local", "compiled": true}
```

The spawned component keeps the default. Read back from a live PIE instance of that class:

```
for comp in child_actor.get_components_by_class(unreal.StaticMeshComponent):
    print(comp.get_name(), comp.get_collision_enabled())
-> Body   <CollisionEnabled.QUERY_AND_PHYSICS: 3>
-> Barrel <CollisionEnabled.QUERY_AND_PHYSICS: 3>
```

Setting `BodyInstance.CollisionProfileName` to `"NoCollision"` instead reports success too and
also does not take. Both calls compiled the Blueprint and the asset saved (size grew), so the
write reached the package — it just is not the write that matters.

The same verb on the *same component* works for plain UPROPERTYs in the same session:
`bOnlyOwnerSee`, `CastShadow`, `bReceivesDecals`, `bUseViewOwnerDepthPriorityGroup`,
`ViewOwnerDepthPriorityGroup`, `AnimClass`, `BoundsScale` all applied and were observable at
runtime. It is specifically the `BodyInstance` sub-struct that does not survive to the instance.

## Why it matters

Collision is derived state. `FBodyInstance::CollisionEnabled` is consumed when the physics state
is created; storing the field by reflection after `AllocateDefaultPins`-equivalent template setup,
with no `SetCollisionEnabled()` call and no physics-state recreation, leaves the stored value and
the live body disagreeing. `UPrimitiveComponent::SetCollisionEnabled` /
`SetCollisionProfileName` are the setters that also call `UpdateCollisionProfile` /
`RecreatePhysicsState`; a raw store skips both. This is the class-template sibling of
`B-collision-write-skips-instance-bodies` (which is `actor.set_component_properties` and ISM
per-instance bodies) and of `B-set-component-properties-no-change-notification` (DONE, the
"store is not the write" pattern) — neither covers `blueprint.scs.set_property`.

## What it cost here

A first-person character whose viewmodel is a skeletal mesh plus a child-actor weapon, both
attached to the camera. With the weapon's cubes still colliding, the capsule's floor trace hit its
own weapon every frame and the pawn **climbed 15 000 cm in a few seconds** while
`CharacterMovement` still reported `MOVE_WALKING` and velocity `(0,0,0)`. Two rounds of
`scs.set_property` reporting success made the problem look like something else entirely; only a
live `get_collision_enabled()` read on the spawned child actor found it.

## Expected

- Route `BodyInstance.CollisionEnabled` / `BodyInstance.CollisionProfileName` (and
  `BodyInstance.ObjectType`, the response channels) through the component's setters on the
  template so the value is real on every spawned instance.
- If the typed route is not available for an SCS template, **fail** rather than report success on
  a store that cannot take effect — the current answer is indistinguishable from a working write.
- Document on `blueprint.scs.set_property` which nested struct paths are honoured; the page
  currently advertises dot-paths generally ("supports dot-paths for nested struct members (e.g.
  'RelativeLocation.X')") with no exceptions listed.

## Workaround used

Author a `BeginPlay` on the child-actor class that calls the real setter:

```
entry event BeginPlay() {
    call SetActorEnableCollision(Target: self, bNewActorEnableCollision: false)
}
```

Verified: after this, the pawn rests at Z 92.15 on a capsule of half-height 90 and stays there.

severity rationale: impact=silent false success on derived state, with a failure that presents as
an unrelated physics bug x reach=any Blueprint whose SCS components need non-default collision
-> High

## History
- `#1-filed` `OPEN` reporter — Hit on UE 5.8 / EAContentExamples58 while building the FPS PLAYER stream's first-person character. Two `blueprint.scs.set_property` calls (`BodyInstance.CollisionEnabled` = `"NoCollision"`, then `BodyInstance.CollisionProfileName` = `"NoCollision"`) on `/Game/FPS/Player/BP_WeaponStub_PlayerTest`'s `Body` and `Barrel` static-mesh components both returned `success:true` with `source:"local"` and `compiled:true`, and the package saved. A live read on the spawned child actor in PIE still reported `CollisionEnabled.QUERY_AND_PHYSICS` for both. The same verb applied `bOnlyOwnerSee`, `CastShadow`, `bReceivesDecals`, `bUseViewOwnerDepthPriorityGroup`, `ViewOwnerDepthPriorityGroup`, `AnimClass` and `BoundsScale` successfully in the same session, so the failure is specific to the `BodyInstance` sub-struct rather than to dot-paths in general. The visible symptom was severe and pointed away from collision: the character capsule climbed from Z 120 to Z 19 599 in seconds while `CharacterMovement` reported `MOVE_WALKING` with velocity `(0,0,0)` — it was stepping onto its own camera-attached weapon every frame. Worked around with a `BeginPlay` calling `SetActorEnableCollision(false)` on the weapon actor, after which the pawn rests correctly at Z 92.15. Related but distinct: `B-collision-write-skips-instance-bodies` (`actor.set_component_properties`, ISM per-instance bodies) and `B-set-component-properties-no-change-notification` (DONE, same "store is not the write" class on a different verb).

## Fix

**Confirmed TRUE by source read.** `FSCSHandlers::SetSCSComponentProperty`
(`Source/PinWright/Private/PinWright_SCSHandlers.cpp`) resolved the dot-path with
`ResolveNestedPropertyPath` and stored through `ApplyJsonValueToProperty` — a raw reflection
write, no setter, no notification. `actor.set_component_properties` already routed the same
fields through `Utils/BodyInstanceCollisionPropertyWrite`; `blueprint.scs.set_property` did not.

**Root cause (UE 5.8 source).** The collision fields of a component template are re-derived
from `CollisionProfileName` on every instance, *after* the archetype values are copied in:
`USCS_Node::ExecuteNodeOnActor` → `AActor::CreateComponentFromTemplate`
(`ActorConstruction.cpp:1112-1153`) → `StaticDuplicateObjectEx` (`:1140`, whose `FlagMask` at
`:1138` strips `RF_ArchetypeObject`) → `ConditionalPostLoad`, run for exactly the non-template
duplicates (`UObjectGlobals.cpp:3152-3159`) → `UPrimitiveComponent::PostLoad` →
`FBodyInstance::FixupData` under a `!IsTemplate()` guard (`PrimitiveComponent.cpp:1810-1821`) →
`LoadProfileData(false)` (`BodyInstance.cpp:4566`, `:4475-4535`) →
`UCollisionProfile::ReadConfig`, which assigns `CollisionEnabled`, `ObjectType` and the whole
response container off the profile (`CollisionProfile.cpp:197-199`). Every
`UPrimitiveComponent` constructor installs `BlockAll` (`PrimitiveComponent.cpp:361`), so the
value re-applied is `QueryAndPhysics` — the reported symptom exactly. The engine setters keep
the invariant a raw store breaks: `SetCollisionEnabled` / `SetObjectType` /
`SetResponseToChannel(s)` all call `InvalidateCollisionProfileName()`
(`BodyInstance.cpp:564-569, :571-593, :675-680, :742-748`), moving the profile to `Custom`,
one of the two names `IsValidCollisionProfileName` rejects (`:4470-4473`).

A second, independent loss path was found and closed with it: `bUseDefaultCollision` on a
`UStaticMeshComponent` makes `OnRegister` → `UpdateCollisionFromStaticMesh` →
`UseExternalCollisionProfile` → `LoadProfileData` re-read the whole collision setup off the
mesh at every registration (`StaticMeshComponent.cpp:812-826, :2175-2189`). The engine clears
that flag from `UStaticMeshComponent::SetCollisionProfileName` (`:2618-2622`) and from none of
the other three setters.

**Design, and why not the obvious one.** `PreEditChange` / `PostEditChangeChainProperty` from
the shared property helper would NOT have fixed this: neither
`UPrimitiveComponent::PostEditChangeProperty` (`PrimitiveComponent.cpp:1541-1643`) nor
`UStaticMeshComponent::PostEditChangeProperty` (`:1951-2022`) has a `BodyInstance` branch — the
details panel's fix-up lives in the Slate customization `FBodyInstanceCustomization`
(`BodyInstanceCustomization.cpp:816-892`), not in a change hook. `Utils/PropertyChangeNotify.h`
also documents, with engine citations, why the chain form and `PreEditChange` were previously
rejected for this plugin. The chosen shape instead reuses the module that already owns these
fields, adding one dot-path entry point that folds the path tail back into the nested-object
shape and delegates — so both verbs resolve to one implementation and cannot drift.
Instance-of-template propagation was verified NOT to be the loss mechanism: the handler already
runs `MarkBlueprintAsStructurallyModified` + `CompileBlueprint`, the compiler preserves SCS
templates (`FKismetCompilerContext::SaveSubObjectsFromCleanAndSanitizeClass`,
`KismetCompiler.cpp:762-779`), and reinstancing rebuilds placed actors from those templates.

Files changed (all under `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\`):
- `Source/PinWright/Private/Utils/BodyInstanceCollisionPropertyWrite.h/.cpp` — new
  `ApplyBodyInstanceCollisionPropertyPath(Component, PropertyPath, Value, ...)`; and
  `ApplyBodyInstanceCollisionProperty` now clears `bUseDefaultCollision` on a
  `UStaticMeshComponent` before replaying the setters.
- `Source/PinWright/Private/PinWright_SCSHandlers.cpp` — new file-local
  `ApplySCSTemplatePropertyValue` tries the collision route then falls back to the plain
  reflection store; used at both write sites (local/CDO template and the ICH override
  template). Response gains `collisionRouted: true` when a collision field actually routed.
- `Source/PinWright/Private/Tests/Blueprint/TestSCSSetPropertyCollision.cpp` — new, 4 tests.
- `Docs/wiki-src/blueprint.scs.md` — new `### blueprint.scs.set_property` section naming which
  nested paths are honoured (the third bullet of Expected).
- `Docs/wiki-src/actor.md` — the `bUseDefaultCollision` note, since the shared helper change
  reaches `actor.set_component_properties` too.

Tests added (`PinWright.blueprint.scs.set_property.*`):
- `CollisionEnabledReachesTheSpawnedInstance` — fixture asserted to START at QueryAndPhysics,
  write, then assert on the **spawned** component, not the template.
- `CollisionProfileNameLoadsTheProfileData` — asserts the template implements the profile it
  names (the raw store wrote the name only; the instance repairs itself via `FixupData`, so
  the template assertion is the load-bearing one here).
- `CollisionWriteClearsUseDefaultCollision` — cube-meshed template with the flag set.
- `NonCollisionBodyFieldKeepsTheReflectionStore` — narrowness guard on `BodyInstance.MassScale`.

**Reviewer verification.** Not compiled and not run here (a separate compile pass follows). Run
`PinWright.blueprint.scs.set_property` and `PinWright.actor.set_component_properties` plus
`PinWright.Actor.*`; the four new tests must pass and no existing collision test may regress.
Live check: `blueprint.scs.set_property` `BodyInstance.CollisionEnabled = "NoCollision"` on a
static-mesh SCS template must return `collisionRouted: true`, and a PIE
`get_collision_enabled()` on a spawned instance must report `NO_COLLISION`. Read the result off
a spawned actor — `blueprint.scs.get` / `scs.json` report template state, which was always right.
