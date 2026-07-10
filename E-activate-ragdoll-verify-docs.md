---
id: E-activate-ragdoll-verify-docs
title: "physics.activate_ragdoll wiki names no way to verify its effect — the sim-state readback property (BodyInstance.bSimulatePhysics) is undocumented (forced an engine-source dive) and the doc never says the RPC's own return (ragdollActive/hasPhysicsAsset/physicsAssetPath) already confirms the setup (agent set PhysicsAssetOverride + re-read, 2 extra RPCs, to make it verifiable)"
status: OPEN
severity: Low
category: ergonomic
tags: [physics, activate_ragdoll, verify-effect-docs, readback, discoverability, docs]
encounters: 2
lastSeen: 2026-07-10T20:08:17.7925570+03:00
---

# `physics.activate_ragdoll` wiki doesn't say how to verify its effect (nor that its own return already confirms it)

`physics.activate_ragdoll` works correctly — `activate=true` flips the
skeletal-mesh component's `BodyInstance.bSimulatePhysics` false→true and
`activate=false` flips it back — but its wiki overlay documents **no way to
independently confirm the flip took**, and no note that the RPC's own return
value already carries the confirmation. On a "knockout and recover" ragdoll
round-trip task (outcome clean, no retries, no python fallback) this forced two
separate detours at verification time. Two facets, one page:
`docs/wiki-src/physics.md`.

## Facet A — the sim-state readback property is undocumented (engine-source dive)

The `physics.activate_ragdoll.md` page lists only the params `actorName` /
`activate` — nothing about which reflected property reads back the resulting
simulation state. To confirm the collapse/reset actually took effect on the
character (rather than trusting the RPC's own `ragdollActive` flag), the agent
had to read the skeletal-mesh component's simulation state, but had to
**reverse-engineer the property name from engine C++**: it grepped
`SkeletalMeshComponentPhysics.cpp` and found `USkeletalMeshComponent::SetSimulatePhysics`
writes `BodyInstance.bSimulatePhysics = bSimulate` unconditionally, then read
that property via `actor.get_component_property` (confirmed ON=true after
activate, OFF=false after deactivate).

Friction note (verbatim): *"I did read one engine header/cpp to confirm
SetSimulatePhysics writes BodyInstance.bSimulatePhysics (the readback property),
which the wiki did not name directly."*

## Facet B — the wiki never says the RPC's own return already confirms the physics-asset setup

`physics.activate_ragdoll`'s return already carries
`ragdollActive` / `hasPhysicsAsset` / `physicsAssetPath` (the call log shows
`activate=true -> ragdollActive:true hasPhysicsAsset:true`), which is exactly the
"is a physics asset present on this component" confirmation. Nothing in the wiki
says so, so to make the component-level physics-asset assignment independently
verifiable the agent instead explicitly set the component's `PhysicsAssetOverride`
= `PA_Mannequin` and re-read it — **2 extra RPCs** (`actor.set_component_properties`
+ a verify `actor.get_component_property`) that the return already made
redundant. The mesh (`SKM_Manny_Nanite`) already ships `PA_Mannequin` and
`property.get` on the mesh already returned it, so no override was needed at all.

Friction note (verbatim): *"the component's EFFECTIVE physics asset (inherited
from the mesh default) is not exposed as a reflected property — only
PhysicsAssetOverride is readable, and it's null when inherited — so to make the
component-level assignment verifiable I set the override explicitly."*

Caveat (from the call-trace analysis): the "PhysicsAssetOverride reads null when
inherited" claim is the agent's UE-semantics **inference** — it never read the
override before setting it, so the trace shows the workaround reasoning, not a
demonstrated null. The effective physics asset comes from
`USkeletalMeshComponent::GetPhysicsAsset()` (a function that falls back to the
mesh's default `PhysicsAsset`), of which the only reflected UPROPERTY is
`PhysicsAssetOverride`.

## What it should do

Add a "Verifying the effect" note to the `### physics.activate_ragdoll` section
of `docs/wiki-src/physics.md`:

1. **Sim-state readback:** after `activate=true`/`false`, read
   `BodyInstance.bSimulatePhysics` on the skeletal-mesh component via
   `actor.get_component_property` to confirm the flip — name the property so no
   one has to dig it out of `SkeletalMeshComponentPhysics.cpp`.
2. **Return already confirms setup:** state that the RPC's return already carries
   `ragdollActive` (the sim-state confirmation) and `hasPhysicsAsset` /
   `physicsAssetPath` (the physics-asset confirmation), so a caller does not need
   to set `PhysicsAssetOverride` explicitly to make the assignment verifiable.
3. **Effective physics asset:** note that a skeletal-mesh component's effective
   physics asset is `GetPhysicsAsset()` (falls back to the mesh's default
   `PhysicsAsset`, readable via `property.get` on the mesh) and that
   `PhysicsAssetOverride` is null when the asset is inherited — so its null
   readback is not a missing asset.

Both facets close a verification detour the agent flagged (one engine-source
dive + two redundant RPCs) on an otherwise-clean run.

## Evidence

Task `physics.activate_ragdoll` (focus), a "knockout and recover" ragdoll
round-trip on a spawned `SKM_Manny_Nanite` (`KO_Manny`): 10 real RPCs, no
is_error, no retries, no hung/crashed calls, no python.execute fallback. The
only friction was at verification: (A) an engine-source grep to name the
`BodyInstance.bSimulatePhysics` readback property, and (B) 2 extra RPCs
(`actor.set_component_properties` PhysicsAssetOverride + verify readback) that
`activate_ragdoll`'s own `hasPhysicsAsset`/`physicsAssetPath` return already made
redundant. Focus method `physics.activate_ragdoll` itself functioned correctly —
the gap is purely documentation of how to verify its effect.

severity rationale: impact=docs/discoverability × reach=rare (ragdoll authoring is
a specialized physics path, not an every-session method) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of a clean `physics.activate_ragdoll` round-trip task (10 RPCs, no retries/errors/python fallback). Two verification detours the wiki forced: (A) the sim-state readback property `BodyInstance.bSimulatePhysics` is undocumented, so the agent grepped `SkeletalMeshComponentPhysics.cpp` to learn `USkeletalMeshComponent::SetSimulatePhysics` writes it; (B) to verify the component's physics asset the agent set `PhysicsAssetOverride`=`PA_Mannequin` + re-read (2 extra RPCs) even though `activate_ragdoll`'s return already reported `hasPhysicsAsset:true`/`physicsAssetPath` and `property.get` on the mesh already returned `PA_Mannequin`. Page to improve: `docs/wiki-src/physics.md`, `### physics.activate_ragdoll` — add a "Verifying the effect" note naming `BodyInstance.bSimulatePhysics`, stating the return already carries `ragdollActive`/`hasPhysicsAsset`/`physicsAssetPath`, and that the effective asset is `GetPhysicsAsset()`/the mesh default (`PhysicsAssetOverride` null when inherited — agent-inferred). Dedup: `grep -rl activate_ragdoll` on the board found no ticket naming this verb; same symptom family as `E-networking-wiki-stub-no-readback-contract` (wiki names no verify/readback contract → source dive) but that ticket is method-scoped to `networking.get_networking_info` (different method/namespace/root cause), so filed separately per the "over-broad prior ticket must not absorb a distinct friction" rule.
- `#2-additional-wrong-readback-name-guess` `OPEN` reporter — Same root cause, new angle from a `physics.setup_ragdoll`-focus task ("make SK_DinoDragon go limp and collapse into a ragdoll; then confirm it's genuinely simulating and not frozen in its anim pose"). Because `docs/wiki-src/physics.md` names no canonical sim-state readback property, the agent (and this iteration's success-check + Prep plan, which both blessed "PhysicsBlendWeight reads ~1.0" as a ragdoll-confirmation datum) reached for `PhysicsBlendWeight` FIRST — `actor.get_component_property {actorName=SK_DinoDragon, componentName=SkeletalMeshComponent0, propertyName=PhysicsBlendWeight}` returned `[NOT_FOUND] Property 'PhysicsBlendWeight' not found on component 'SkeletalMeshComponent0'`, while sibling reads on the same component resolved (`BodyInstance.bSimulatePhysics` → false, `PhysicsAssetOverride` → null). The Judge's replay confirmed the NOT_FOUND is CORRECT — `PhysicsBlendWeight` is not a reflected UPROPERTY, so the clean NOT_FOUND is accurate, not misleading (no defect). Agent recovered instantly by falling back to `BodyInstance.bSimulatePhysics`. This is additional evidence for Facet A: the "Verifying the effect" note should not just NAME `BodyInstance.bSimulatePhysics` but explicitly steer callers AWAY from `PhysicsBlendWeight` (a canonical USkeletalMeshComponent anim/physics-blend field that is NOT reflected and NOT readable via `actor.get_component_property`), and the iteration's own success-check phrasing ("PhysicsBlendWeight reads ~1.0") should be corrected to `bSimulatePhysics=true`. NOTE: the same task also hit a suspected editor crash on `asset.save SK_DinoDragon` after `physics.setup_physics_simulation assignToMesh=true` — the per-finding Judge replayed it 5x + the full sequence twice (all `saved:true`, editor healthy) and classed it `nonrepro`/transient-environmental, so no bug ticket was filed for that. `encounters` 1→2.
