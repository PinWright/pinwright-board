---
id: B-scs-set-property-bodyinstance-collision-silent-noop
title: "blueprint.scs.set_property on BodyInstance.CollisionEnabled / BodyInstance.CollisionProfileName returns success but the spawned component keeps QUERY_AND_PHYSICS — a class-template collision write that never reaches the instance"
status: OPEN
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
