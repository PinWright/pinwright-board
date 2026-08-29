---
id: F-component-collision-channel-write
title: "No verb writes collision state on a NAMED COMPONENT — `actor.set_collision` takes `{actorName, collisionEnabled}`, touches only the root primitive and says so in its own summary, so `ECC_Pawn -> Ignore` on one HISM of a 15-component scatter holder is unreachable and the `python.execute` fallback it forces is measured broken on exactly the component class the plugin's own scatter recipe produces"
status: OPEN
severity: Medium
category: feature
tags: [actor, set_collision, set_component_properties, components, collision, collision-channel, collision-profile, body-instance, ism, hism, pawn, missing-verb, level-building, vegetation]
encounters: 1
lastSeen: 2026-08-29T21:55:00+03:00
---

# The only collision write is a boolean on the root

`actor.set_collision` (`Private/Handlers/Actor/ActorPropertyHandler.cpp:182`) declares two
parameters — `actorName` and `collisionEnabled` — and its registered summary states its own limit:

> "Toggle collision on the actor's root primitive component between QueryAndPhysics (enabled) and
> NoCollision (disabled). **Affects only the root; component-level collision channels are not
> touched.**"

The body matches: `Actor->GetRootComponent()`, `Cast<UPrimitiveComponent>`, then
`SetCollisionEnabled(QueryAndPhysics)` or `(NoCollision)` (`ActorPropertyHandler.cpp:207-215`).
There is no second write anywhere. `actor.set_component_properties` can be handed
`{"BodyInstance": {...}}` and will store it by raw reflection, which is not a collision write and is
its own defect (`B-collision-write-skips-instance-bodies`). No verb takes a channel name, a response
value, a profile name, or an object type.

## What that costs, measured

Build: the editor's **10:44** PinWright build; source read at HEAD `962275fa`. Level
`/Game/Maps/PW_VegetationTest`. Scripts and receipts:
`X:/src/unreal/EAContentExamples58/dev/grasscollide/`. Write-up:
`Docs/map/vegetation-agent-brief.md` § *Every plain HISM built by the zone-F recipe blocks the Pawn
channel*.

The user reported "why does grass have player collision?". The cause is the plugin's own documented
scatter recipe: `actor.add_component
{componentType:"HierarchicalInstancedStaticMeshComponent"}` comes up **`QueryAndPhysics` /
`BlockAllDynamic`**, i.e. `ECC_Pawn = Block`, which nobody asked for and which is the exact inverse
of the engine foliage path (`NoCollision` / `NO_COLLISION`). 5888 instances of groundcover, ferns and
flowers in `ZoneF_Veg` were built that way.

The needed edit is one line of intent — *stop these twelve components blocking the Pawn channel,
change nothing else* — and it is not expressible:

- **`actor.set_collision` cannot reach the components.** `ZoneF_Veg` is a bare `AActor` holder
  carrying **15** HISM components; none is the root. The wiki's own holder recipe
  (`Saved/PinWright/wiki/level-building.instancing-and-scatter.md`) produces exactly this shape.
- **`actor.set_collision` is also the wrong axis even where it does reach.** It is
  enabled/disabled. Turning the whole component off would have destroyed the level's ground probes:
  the same components must keep **Visibility / WorldStatic / WorldDynamic / Camera at Block** —
  `Docs/map/vegetation-zone-f.md` § *The exclusion contrast, reproducible* is a shipped test fixture
  that depends on a HISM answering `any_solid`, and `spatial.ground_actors` /
  `spatial.verify_grounding` are run against these components. Per-channel is the whole point.
- **The fallback was `python.execute`**, calling `comp.set_collision_response_to_channel(ECC_Pawn,
  ECR_IGNORE)` on each component — which returned cleanly, read back `ECR_IGNORE`, moved the profile
  to `Custom`, and left 7 of 12 components still blocking a pawn capsule sweep. Full measurement and
  mechanism: **`B-collision-write-skips-instance-bodies`**. Cost: three engine files read
  (`PrimitiveComponentPhysics.cpp`, `InstancedStaticMesh.cpp`,
  `InstancedMeshComponentBodies.cpp`), an undocumented `SetCollisionEnabled(NoCollision)` ->
  `(QueryAndPhysics)` toggle to make the write take, and a 128-sample capsule-sweep harness written
  from scratch to prove it, because no readback could distinguish the two states
  (`E-get-components-omits-collision-state`).

## Ask

A component-scoped collision verb, symmetric with `actor.set_component_properties`' addressing:

    actor.set_component_collision {
      actorName, componentName,
      collisionEnabled?:  "NoCollision" | "QueryOnly" | "PhysicsOnly" | "QueryAndPhysics",
      collisionProfile?:  <profile name>,          // mutually exclusive with the two below
      objectType?:        <channel name>,
      channelResponses?:  { "Pawn": "Ignore", "Visibility": "Block", ... }
    }

Three properties it must have, each earned above:

1. **Per-channel, not per-component-on/off.** The measured case needs Pawn changed and four other
   channels untouched.
2. **It must route through the engine setters and then propagate to the instance bodies**, per
   `B-collision-write-skips-instance-bodies` § *Ask*, and report what it refreshed. A verb that only
   calls `SetCollisionResponseToChannel` ships that ticket's defect on day one — measured, not
   predicted.
3. **The channel vocabulary is shared.** Naming a channel needs a string ->
   `ECollisionChannel` map, and the four copies of that map that already exist in the tree accept
   only `visibility` / `camera` / `worldstatic` / `worlddynamic` — no `Pawn`, no
   `ECC_GameTraceChannel*`. See `F-trace-channel-vocabulary-incomplete`; the fix for that ticket is
   this ticket's prerequisite, and the same map should serve both rather than becoming a fifth copy.

The matching **read** is `E-get-components-omits-collision-state`, filed separately because it lands
on an existing enumeration verb rather than on a new one.

## Also worth fixing on the verb that exists

`actor.set_collision`'s two guards are silent. `Actor->GetRootComponent()` returning null, and
`Cast<UPrimitiveComponent>` failing on a `DefaultSceneRoot`, both fall through to an unconditional
`SendSuccess` that echoes the **requested** `collisionEnabled` back
(`ActorPropertyHandler.cpp:207-221`). So every wiki-recipe HISM holder — a bare `AActor` whose root
is a plain `USceneComponent` — answers `success` with `collisionEnabled: true` and writes nothing.
**Source-only; not exercised in this pass** (the verb was never called, precisely because it cannot
reach a named component). Filed separately as `B-set-collision-nonprimitive-root-silent-success`
rather than folded in here, because it is a silent-false-success bug and this is a missing-capability
request; merging them would work the bug at this ticket's severity.

## Not a duplicate of

- **`B-collision-write-skips-instance-bodies`** (OPEN, High, filed from this pass) — its partner.
  That one is why the write did not work once it got there; this one is why there was nothing to call.
  Disjoint fixes: propagating filter data to instance bodies is not a new verb, and a new verb does
  not propagate anything by itself. Whoever takes either should read both.
- **`B-shape-extent-stale-physics`** (IN-REVIEW, High) — uses `actor.set_collision {false}` then
  `{true}` as its documented workaround for forcing a physics rebuild, and notes it invalidates the
  profile to `Custom` as a side effect. That is the board already using this verb as a hammer for
  want of a scalpel, and it is corroborating evidence rather than an overlap.
- **`E-get-component-property-no-subfield-select`** (DONE, Medium) — the read half of the same gap,
  narrowed to one dotted field. Shipped. Its own workaround note ("read collision via
  `actor.set_collision` round-trip") is direct evidence that the write side has never been asked for.
- **`B-declared-param-guard-blind-spots`** / **`B-verbs-read-undeclared-parameters`** (both
  IN-REVIEW, High) — both name `actor.set_collision`, for reading `collision_enabled` /
  `actor_name` off the raw payload without declaring them. Schema hygiene on the same verb, nothing
  about what it writes or how far it reaches.
- **`E-ground-preset-excludes-only-foliage-actors`** (DONE, High) — added `excludeComponentClasses`
  so a ground probe can *skip* a component class. That is a query-side filter; this is a write.

## Severity

**Medium**, on the rubric's soft-blocker band: *"doable, but only via a documented workaround, a
source dive, or many extra calls."* It was done — but through `python.execute`, after reading three
engine source files, and only correctly after discovering an undocumented two-step toggle whose
necessity is invisible from every field the RPC surface returns.

**Bump-up to High declined, twice over.** (a) The "hard blocker with no workaround" band does not
apply: `python.execute` is part of this surface and the edit landed. (b) The stronger argument — that
the workaround is *measured broken* on instanced components — is real, but that impact is
`B-collision-write-skips-instance-bodies`' to carry at High, and pricing it into both tickets would
count one defect twice in the picker's ordering.

**Bump-down to Low declined.** Low is "pure friction: docs, discoverability, naming". This is not
reachable by any typed verb at all, and the thing it blocks — deciding what a player can walk
through — is not cosmetic. Nor is it a rare edge path: every scatter built the way the plugin's own
wiki teaches comes up blocking every channel, so every such scatter needs this write eventually.

## History
- `#1-no-component-scoped-collision-write` `OPEN` reporter — Filed from a collision-and-seating fix
  pass on `/Game/Maps/PW_VegetationTest` (editor build 10:44; plugin source HEAD `962275fa`). The
  task was to stop 5527 instances of groundcover, fern and flower blocking a walking player while
  keeping every other channel Block, on 12 HISM / foliage-ISM components of two holder actors. No
  typed verb can address a named component's collision at all: `actor.set_collision`
  (`ActorPropertyHandler.cpp:182`) declares `{actorName, collisionEnabled}`, casts the root to
  `UPrimitiveComponent` and toggles `QueryAndPhysics`/`NoCollision`, and its own summary states
  "Affects only the root; component-level collision channels are not touched." The holders are bare
  `AActor`s with 15 and dozens of HISM children respectively, so the root is not even a primitive.
  The pass fell back to `python.execute`; that route is separately broken on instanced components
  and is filed as `B-collision-write-skips-instance-bodies`. Dedup: searched the board for
  `SetCollisionResponseToChannel`, `SetCollisionProfileName`, `BodyInstance`, `ECC_`,
  `ECollisionChannel`, `collision_response`/`collisionResponse`,
  `collision_profile`/`collisionProfileName`, `per-channel`, `actor.set_collision`, `set_collision`,
  `BlockAll`/`NoCollision`/`QueryAndPhysics`, and a title-only scan for `collision`. Nothing asks for
  a per-channel response write, a profile write, a `BodyInstance` write, or a non-root collision
  target. The four board files that mention `SetCollisionProfileName`/`SetCollisionEnabled` cite
  them as engine source inside tickets about something else, and the closest neighbour,
  `B-shape-extent-stale-physics`, *uses* this verb as a workaround without asking for a better one.
  Recorded and split out rather than folded in: `actor.set_collision`'s two silent guards
  (`ActorPropertyHandler.cpp:207-221`) report success on an actor whose root is not a primitive —
  `B-set-collision-nonprimitive-root-silent-success`.
