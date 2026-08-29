---
id: B-collision-write-skips-instance-bodies
title: "A collision write on an ISM/HISM lands on the component's own `BodyInstance`, which owns no shapes — the per-instance bodies keep the filter data they were COPIED at creation, so `ECC_Pawn -> Ignore` reads back `ECR_IGNORE` with profile `Custom` on all 12 components while a pawn-profile capsule sweep still names 7 of them as the blocker; only a `SetCollisionEnabled(NoCollision)` -> `(QueryAndPhysics)` toggle made it real, and `actor.set_component_properties`, the only verb that can express the write at all, stores `BodyInstance` by raw reflection with no setter and no physics-state recreation"
status: OPEN
severity: High
category: bug
tags: [actor, set_component_properties, components, collision, body-instance, ism, hism, instanced-static-mesh, physics-state, filter-data, silent-false-success, derived-state, corroborated-readback, pawn, vegetation, level-building]
encounters: 1
lastSeen: 2026-08-29T21:55:00+03:00
---

# The component's body is not the instances' bodies, and only one of them is writable

A `UInstancedStaticMeshComponent` draws and collides through **per-instance** `FBodyInstance`s. Its
own inherited `UPrimitiveComponent::BodyInstance` owns no shapes at all — it is a **template** that
is copied into each instance body once, at body creation, and never consulted again. Every collision
write the engine and this plugin can reach goes to the template.

So a caller who sets `ECC_Pawn -> Ignore` on a scatter gets: a clean return, a read-back of
`ECR_IGNORE`, a profile that flips `BlockAllDynamic -> Custom` — and a world in which the pawn is
still blocked by that exact component.

## Measured

**Build:** the editor was running the **10:44** PinWright build; the write was issued through
`python.execute` calling the engine's own `set_collision_response_to_channel`, so the plugin build is
not load-bearing for this half — the mechanism is engine-side. Level
`/Game/Maps/PW_VegetationTest`, ~30,000 instances across 147 collision-bearing components. Scripts
and raw receipts: `X:/src/unreal/EAContentExamples58/dev/grasscollide/` (untracked in the host
project). Full write-up: `Docs/map/vegetation-agent-brief.md` § *Every plain HISM built by the
zone-F recipe blocks the Pawn channel*.

**The write, on 12 HISM / foliage-ISM components (5527 instances)** —
`dev/grasscollide/out/fix_report.json`, one row per component, each field a `[before, after]` pair:

    "pawn":             ["ECR_BLOCK", "ECR_IGNORE"]
    "visibility":       ["ECR_BLOCK", "ECR_BLOCK"]
    "collisionEnabled": ["QUERY_AND_PHYSICS", "QUERY_AND_PHYSICS"]
    "profile":          ["BlockAllDynamic", "Custom"]
    "status":           "OK"

12 of 12 `OK`. The `after` column is a real read-back through
`GetCollisionResponseToChannel(ECC_Pawn)` and `GetCollisionProfileName()`, not an echo of the
request.

**The behaviour, immediately after that write** — a capsule sweep on the stock `Pawn` profile
(r 34, half-height 88, 600 cm horizontal at ground+90,
`SystemLibrary.capsule_trace_multi_by_profile`) at 8 frozen sites,
`dev/grasscollide/out/sweep_before.json`. Blocking hits, by component:

| component | simple sweep | complex sweep |
|---|---:|---:|
| `HISM_ZF_GC6` | — | 2 |
| `HISM_ZF_GC24` | — | 2 |
| `HISM_ZF_GC52` | — | 1 |
| `HISM_ZF_Fern8` | — | 1 |
| `HISM_ZF_Flower` | — | 1 |
| `HISM_ZF_Thicket` | 1 | 1 |
| `FoliageInstancedStaticMeshComponent_44` | 1 | 1 |

**7 of the 12 components that had just reported `ECR_IGNORE` were still blocking the Pawn channel.**

**What made it real:** `SetCollisionEnabled(NoCollision)` then `SetCollisionEnabled(QueryAndPhysics)`
on each component — nothing else changed, no response was rewritten
(`dev/grasscollide/out/refresh_report.json`, `status: "REFRESHED"`, `pawn: "ECR_IGNORE"`,
`visibility: "ECR_BLOCK"`, `collisionEnabled: "QUERY_AND_PHYSICS"` on every row). The same 8-site
sweep re-run afterwards (`sweep_after.json`) reports **zero** hits on any of the twelve; the only
non-terrain blockers left are `HISM_ZF_HillTree` and a `StaticMeshComponent0` (a fallen log), both
deliberately left blocking. 120 further sweeps over 8 ground-following traverses across four zones
(`traverse.json`) agree: one blocker level-wide, and it is the log.

## Mechanism, re-derived at UE 5.8 (`C:/UE_5.8/Engine/Source/Runtime/Engine/`)

**The write reaches one body, and it is the wrong one.**

    Private/PrimitiveComponentPhysics.cpp:1352   void UPrimitiveComponent::SetCollisionResponseToChannel(...)
    Private/PrimitiveComponentPhysics.cpp:1354       if (BodyInstance.SetResponseToChannel(Channel, NewResponse))
    Private/PrimitiveComponentPhysics.cpp:1356           OnComponentCollisionSettingsChanged();

**The notification cannot fix it, and ISM/HISM does not override it.**
`OnComponentCollisionSettingsChanged` (`Private/PrimitiveComponentPhysics.cpp:1434-1459`) clears the
overlap-skip cache, calls `UpdateOverlaps()`, refreshes navigation relevancy and broadcasts a
delegate. It touches no body and no filter data. The virtual is declared at
`Classes/Components/PrimitiveComponent.h:2963`, and neither `Private/InstancedStaticMesh.cpp` nor
`Private/HierarchicalInstancedStaticMesh.cpp` overrides it.

**The instance bodies are a one-time copy of the template.**

    Private/InstancedStaticMesh.cpp:2868           void UInstancedStaticMeshComponent::CreateAllInstanceBodies()
    Private/InstancedStaticMesh.cpp:2864               InitInstanceBody(InstanceIdx, InstanceBodyInstance, &BodyInstance, false, GetBodySetup());
    Private/InstancedMeshComponentBodies.cpp:33    void FInstancedMeshComponentBodies::InitInstanceBody(FBodyInstance* Body, FBodyInstance* ReferenceBody, ...)
    Private/InstancedMeshComponentBodies.cpp:47        Body->CopyBodyInstancePropertiesFrom(ReferenceBody);

`CopyBodyInstancePropertiesFrom` runs exactly once per instance, from `OnCreatePhysicsState`
(`Private/InstancedStaticMesh.cpp:2909`, `CreateAllInstanceBodies()` at `:2922`). After that the
template and the shapes are independent.

**Why the enable/disable toggle works.** `SetCollisionEnabled`
(`Private/PrimitiveComponentPhysics.cpp:1376`) calls `EnsurePhysicsStateCreated()` at `:1384`, which
is `if (IsPhysicsStateCreated() != ShouldCreatePhysicsState()) { RecreatePhysicsState(); }`
(`Private/Components/PrimitiveComponent.cpp:998-1006`). Going to `NoCollision` destroys the state,
going back to `QueryAndPhysics` recreates it — and the recreate re-runs `CreateAllInstanceBodies`,
which re-copies the template that now says `Ignore`. It is not a "refresh"; it is a rebuild that
happens to re-read the value.

## The engine already does this propagation, on the sibling path

`UInstancedStaticMeshComponent` **overrides** `OnActorEnableCollisionChanged` for exactly this
reason:

    Private/InstancedStaticMesh.cpp:5023   void UInstancedStaticMeshComponent::OnActorEnableCollisionChanged()
    Private/InstancedStaticMesh.cpp:5025       Super::OnActorEnableCollisionChanged();
    Private/InstancedStaticMesh.cpp:5027       for (FBodyInstance* InstanceBody : *InstancePhysicsBodies)
    Private/InstancedStaticMesh.cpp:5031               InstanceBody->UpdatePhysicsFilterData();

So the engine knows the filter data has to be pushed down, does it on the actor-level toggle, and
does not do it on the per-channel response path. That loop is also the cheapest correct fix
available to this plugin — see § *Ask*.

## The PinWright half — SOURCE-ONLY, at HEAD `962275fa`

Not measured. No PinWright verb was used for the collision write in the pass above, because none can
express it (`F-component-collision-channel-write`). What source says about the one verb that could:

`actor.set_component_properties` (`Private/Handlers/Actor/ComponentHandler.cpp:228`) can be handed
`{"BodyInstance": {...}}`, and it will take it:

- `UPrimitiveComponent::BodyInstance` is a reflected `UPROPERTY(EditAnywhere, BlueprintReadOnly)`
  (`C:/UE_5.8/.../Classes/Components/PrimitiveComponent.h:1443-1444`), so
  `ComponentClass->FindPropertyByName` (`ComponentHandler.cpp:349`) resolves it.
- `ApplyJsonValueToProperty`'s `FStructProperty` branch takes a JSON object and recurses per
  sub-property (`Private/Utils/PropertyImport.cpp:848` the branch, `:858` the sub-property lookup,
  `:866` the recursion), so `CollisionResponses` / `CollisionProfileName` / `CollisionEnabled` are
  all reachable by name.
- **Neither interceptor applies.** The handler routes only StaticMesh/SkinnedAsset
  (`:362-364`) and shape extents (`:370-374`) to typed setters, and special-cases only Mobility
  (`:294-325`) and SimulatePhysics (`:333-347`). A `BodyInstance` write falls through to the raw
  reflection store at `:382`.
- **The notification is inert here.** `PinWright::NotifyPropertyChanged` (`:392`) fires a non-chain
  `PostEditChangeProperty`, and `UPrimitiveComponent::PostEditChangeProperty`
  (`C:/UE_5.8/.../Private/Components/PrimitiveComponent.cpp:1541-1643`) branches on
  `LDMaxDrawDistance`, `bAllowCullDistanceVolume`, `bNeverDistanceCull`, `Mobility`,
  `MinDrawDistance`, `bLightAttachmentsAsGroup`, `FirstPersonPrimitiveType` and
  `CustomPrimitiveData`. **There is no `BodyInstance` branch.**
- The only post-write refresh is `MarkRenderStateDirty()` + `UpdateComponentToWorld()` (`:410-411`).
  Neither touches Chaos filter data. `RecreatePhysicsState` is not called anywhere in the file.

So through this verb a collision write is worse than the engine route measured above: the engine
route at least reaches the template. The reflection store reaches the template's *serialised* form
and never calls `SetResponseToChannel`, so even a plain (non-instanced) `UStaticMeshComponent` keeps
the old filter data until something else recreates its body. The response would still report
`applied: ["BodyInstance"]`.

## Ask

1. **Propagate.** On a collision write to a `UInstancedStaticMeshComponent`, loop
   `InstancePhysicsBodies` and call `UpdatePhysicsFilterData()` — the exact body of the engine's own
   `OnActorEnableCollisionChanged` override (`InstancedStaticMesh.cpp:5027-5033`). It is cheaper
   than `RecreatePhysicsState()`, it cannot drop the instance array, and it is the engine's own
   answer to this question on the adjacent path. For a non-instanced primitive the equivalent is
   `BodyInstance.UpdatePhysicsFilterData()`, or `RecreatePhysicsState()` if the write changed
   `CollisionEnabled`.
2. **Report it.** A `physicsStateRecreated` / `instanceBodiesRefreshed` count on the response, so a
   caller can tell a write that took from one that did not. **The precedent already ships:**
   `static_mesh.set_collision_complexity` recreates the physics state of every affected component
   (`Private/Handlers/Asset/StaticMeshSetCollisionComplexityHandler.cpp:128`, counter `:129`) and
   publishes `componentsRecreated` (`:150`). That is the shape wanted here, on the component verb.
3. **Route the write, don't just store it.** `BodyInstance` sub-fields should go through
   `SetCollisionResponseToChannel` / `SetCollisionProfileName` / `SetCollisionEnabled` the way
   StaticMesh already goes through `SetStaticMesh` — the third option
   `B-shape-extent-stale-physics` § *Fix* lists, applied to the property family that ticket's
   § *Scope* names and leaves unenumerated. **What this ticket adds is that for an ISM/HISM the
   typed setter is measured insufficient on its own**: `SetCollisionResponseToChannel` *is* the
   engine's setter, and it left 7 of 12 components blocking. Routing without (1) ships the same lie
   through a better-looking code path.

## Not a duplicate of

- **`B-shape-extent-stale-physics`** (IN-REVIEW, High) — **the parent, and this is filed as a SPLIT
  rather than as a return**, on the precedent of `B-foliage-paint-does-no-ground-projection` `#4`.
  That ticket is `UShapeComponent` extents: same verb, same failure geometry (write lands, reads
  back, trace unchanged), and its § *Scope* explicitly says "collision-geometry fields on primitives
  whose setters call `RecreatePhysicsState()`" are reachable today and deliberately not enumerated.
  Its shipped remedy shape is "route to the typed setter". Returning it to `OPEN` would put the
  picker back onto work already done for shape extents; the two facts it does not carry are that the
  typed setter is **not sufficient** for an instanced component, and that the propagation loop the
  engine already wrote is the fix. Whoever takes either should read both. Note also that its own
  documented workaround — force a rebuild with `actor.set_collision {false}` then `{true}` — is the
  same toggle this ticket measured, which is further evidence the verb is being used as a
  physics-state hammer for want of anything else.
- **`B-set-component-properties-no-change-notification`** (DONE, High) — landed the non-chain
  `PostEditChangeProperty`. Measured here as insufficient for collision, because
  `UPrimitiveComponent::PostEditChangeProperty` has no `BodyInstance` branch to run. Its `#2`
  deliberately rejected `PreEditChange`/`FComponentReregisterContext`, which is precisely why the
  physics half is still stale.
- **`F-component-collision-channel-write`** (OPEN, filed from this pass) — that there is no verb for
  a per-channel collision write at all. Disjoint: that one is why the pass had to use
  `python.execute`; this one is why the write did not work when it got there. A fixer could land
  either without the other, and a new write verb that skips this ticket's propagation ships broken
  on day one.
- **`F-ism-per-instance-transforms`** (IN-REVIEW, High) — the same component class and the same
  "stores silently, moves nothing" shape on the *transform* family, already fixed. Its
  `InstancedMeshUtils::FinishInstanceWrites` (`Private/Handlers/Actor/InstancedMeshUtils.h:195`,
  `MarkRenderStateDirty` `:201`, HISM `BuildTreeIfOutdated(false, true)` `:207`, `MarkPackageDirty`
  `:210`) is the existing per-batch post-write helper any collision fix should sit beside — but it
  is a render/tree helper and touches no physics body, so it is not the fix.
- **`E-get-component-property-no-subfield-select`** (DONE, Medium) — reading
  `BodyInstance.CollisionEnabled` without spilling the struct. Read side, and shipped.
- **`B-property-set-object-hop-notification-noop`** (IN-REVIEW, High) — the generic
  property-notification hole on `property.set`. Different verb, different mechanism (an object hop
  mis-targeting the event); here the event fires on the right object and there is simply nothing in
  the override for it to do.

## Same shape as

The enumeration lives on `B-foliage-paint-does-no-ground-projection` § *Same shape as* and is not
restated. This is a **full** member of the sub-class `B-property-set-object-hop-notification-noop`
identifies: the misleading signal is **corroborated** — the usual defence, read it back, is the one
that fails hardest, because the read walks to the same memory and memory is the half that is right.

It is the fourth member found this session of the narrower family *the write lands, the read-back
confirms it, the behaviour does not change*, after `save_asset(only_if_is_dirty)` (host-project
`CLAUDE.md`), `property.set` through an object hop, and `landscape.get_grass_varieties` returning
copies (`E-python-get-editor-property-returns-live-view`). What this one adds is that the
corroboration is **three-fold and includes a derived field**: the response, the per-channel
read-back, *and* the profile name flipping to `Custom`, which is the engine itself asserting that a
custom response set is in force.

## Severity

**High**, on the rubric's silent-false-success band, with both reach modifiers declined.

**Impact = High, not Critical.** Critical is "editor crash, or a write that corrupts or loses asset
data". Nothing crashes and nothing is lost: the level's stored `BodyInstance` genuinely carries the
new value and a reload materialises it. What is wrong is the live Chaos filter data, and the caller
is told three times that it is right.

**Bump-up to Critical declined** for the reason above — a non-corrupting defect promoted into the
picker's pre-emption band would mis-order the queue against the rubric's own impact definition.

**Bump-down to Medium declined.** The rubric's bump-down is for "a rare edge path". An ISM/HISM is
not one: it is the plugin's own documented scatter primitive
(`Saved/PinWright/wiki/level-building.instancing-and-scatter.md`), `actor.add_component
{componentType:"HierarchicalInstancedStaticMeshComponent"}` is the recipe the wiki teaches, and that
recipe comes up `QueryAndPhysics` / `BlockAllDynamic` — so every scatter built the documented way
starts out blocking every channel, and the only lever for undoing that is the one measured broken
here. Medium's own definition ("doable, but only via a documented workaround") also does not fit: the
workaround exists but is undocumented and undiscoverable, and a caller cannot reach for a workaround
they have no way to know they need, because every signal the RPC surface can return agrees with the
write.

## History
- `#1-instance-bodies-keep-stale-filter-data` `OPEN` reporter — Filed from a collision-and-seating
  fix pass on `/Game/Maps/PW_VegetationTest`, against the editor's **10:44** PinWright build for the
  measured half and source-read at HEAD `962275fa` for the PinWright half. User report was "why does
  grass have player collision?". Diagnosis: `actor.add_component
  {componentType:"HierarchicalInstancedStaticMeshComponent"}` comes up `QueryAndPhysics` /
  `BlockAllDynamic`, the exact inverse of the engine foliage path (`NoCollision` /
  `NO_COLLISION`), so 5888 groundcover/fern/flower instances built by the wiki's own holder recipe
  blocked the Pawn channel. Setting `ECC_Pawn -> Ignore` on the 12 offending components returned
  `OK` 12/12 with a real read-back of `ECR_IGNORE` and `profile` moving `BlockAllDynamic -> Custom`;
  a pawn-profile capsule sweep at 8 frozen sites taken immediately afterwards still named 7 of those
  12 as blocking hits. A `SetCollisionEnabled(NoCollision)` -> `(QueryAndPhysics)` toggle, changing
  no response, cleared all 7; the re-run sweep and 120 further sweeps over 8 traverses across four
  zones then found exactly one non-terrain pawn blocker level-wide, a fallen log that should block.
  Every mechanism line in the body was re-derived against `C:/UE_5.8/` rather than taken from the
  field report, and two claims in that report did not survive: the response write was described as
  leaving the sweep "byte-identical", which cannot be asserted because the pre-write sweep was never
  taken — what is measured is that the **post-write** sweep still names the HISM, which is the
  defect either way; and the level was described as having "18 components blocking Visibility while
  ignoring Pawn", where the census (`dev/grasscollide/out/census_after.json`, 147 collision-bearing
  rows) says **12**, all of them components this pass itself diverged. Dedup: searched the board for
  `SetCollisionResponseToChannel`, `SetCollisionProfileName`, `BodyInstance`, `RecreatePhysicsState`,
  `physicsStateRecreated`, `ECC_`, `ECollisionChannel`, per-channel/collision-response/collision-
  profile in both casings, and every `B-set-component-properties-*` file. `physicsStateRecreated`
  has zero occurrences board-wide. The four existing `SetCollisionProfileName`/`SetCollisionEnabled`
  mentions are incidental engine-source citations inside tickets about something else. Nothing owns
  an instanced-component collision write. Recorded but deliberately NOT filed: the engine's
  `FFoliageStaticMesh::UpdateComponentSettings` copies the foliage *type's* `BodyInstance` onto the
  component unconditionally (`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:1822`,
  definition `:1589`) from `NotifyFoliageTypeChanged` (`:1454`), `CreateNewComponent` (`:1536`) and
  `DetectFoliageTypeChangeAndUpdate` (`:5095`), so a component-level collision fix on a foliage ISM
  reverts if the type asset is edited — engine behaviour with no PinWright residue, recorded in the
  host project's `Docs/map/vegetation-agent-brief.md` instead, on the precedent of that project's
  `Docs/map/vegetation-findings-dossier.md` § H.
