---
id: E-actor-set-component-props-identifier-friction
title: "actor.set_component_properties (and get_component_property/add_component/remove_component) still take bare actorName — reject actorPath — and COMPONENT_NOT_FOUND lists no available component names"
status: OPEN
severity: Low
category: ergonomic
tags: [actor, param-alias, actorname, actorpath, set-component-properties, component-name, error-message, discovery]
encounters: 1
lastSeen: 2026-07-04T00:00:00Z
---

# Component-mutation verbs lag the actorPath-alias migration and their COMPONENT_NOT_FOUND dead-ends discovery

Two independent frictions on `actor.set_component_properties`, hit back-to-back in one task:

1. **Bare `actorName`, no `actorPath`/`objectPath` alias.** The spec declares
   `RPC_PARAM_REQ("actorName", ...)` (`ComponentHandler.cpp:172`) and the body reads
   `Ctx.GetString(TEXT("actorName"))` (`:180`) — no `ActorNameParamUtils`. So the wire
   validator rejects the `actorPath` key that `actor.spawn`/`duplicate` return with
   `MISSING_REQUIRED_PARAM 'actorName'`. Its sibling readers were already migrated to the
   shared path-shaped alias set: `actor.get_components` (`:325`) and `actor.set_transform`
   (`:30`) call `ActorNameParamUtils::ActorNameParamReq`. The four component-mutation verbs
   were left behind — `set_component_properties` (`:172`), `get_component_property` (`:471`),
   `add_component` (`:38`), `remove_component` (`:423`) are all bare `actorName`.
2. **COMPONENT_NOT_FOUND enumerates nothing.** After a case-insensitive name match fails,
   `set_component_properties` returns bare `"Component not found"` (`:216`) with no list of
   the actor's real component names, forcing an `actor.get_components` round-trip to learn
   that the instance is `StaticMeshComponent0`, not the class-ish `StaticMeshComponent`.
   Same dead-end on `get_component_property` (`:531`) and `remove_component` (`:463`).

Not a duplicate: `E-actor-verbs-reject-actorpath-slot` / `E-effect-actor-name-slot-vs-actorname`
(both IN-REVIEW) own the alias *family* but deliberately scoped their fix to reader/transform
verbs — the component verbs are an uncovered follow-up. `E-property-route-no-component-path-discovery`
(IN-REVIEW) recommends these verbs but does not fix their errors; `E-set-niagara-param-error-omits-available-params`
is the same enumerate-in-error pattern for Niagara params.

**Workaround:** call `actor.get_components{actorName:...}` first to read the exact component
instance name, and pass `actorName` (never `actorPath`) to the component verbs.
**Fix:** migrate the four component verbs to `ActorNameParamUtils::ActorNameParamReq` +
`RequireActorName`/`ResolveActorName` (the established pattern), and have COMPONENT_NOT_FOUND
enumerate the actor's actual component names (the loop already walks `Found->GetComponents()`).

## History
- `#1-initial-repro` `OPEN` reporter — Source-verified against `ComponentHandler.cpp` (set_component_properties `:170-320`: spec `:172` bare `actorName`, read `:180`, COMPONENT_NOT_FOUND `:216` with no name list; siblings `add_component:38`, `get_component_property:471/531`, `remove_component:423/463` identical). Confirmed the alias inconsistency: `actor.get_components` (`:325`) and `actor.set_transform` (`ActorTransformHandler.cpp:30`) already use `ActorNameParamUtils::ActorNameParamReq`, so `actorPath` resolves there but hard-fails `MISSING_REQUIRED_PARAM 'actorName'` on the component-mutation verbs. Repro (verbatim from task): (1) `set_component_properties{actorPath:"/Game/_FabGallery/L_Render...StaticMeshActor_0", componentName:"StaticMeshComponent", properties:{OverrideMaterials:[...]}}` → `[MISSING_REQUIRED_PARAM] 'actorName'`; (2) retry `{actorName:"RTArrow", componentName:"StaticMeshComponent", ...}` → `[COMPONENT_NOT_FOUND] Component not found`; (3) `actor.get_components{actorName:"RTArrow"}` revealed `"StaticMeshComponent0"`, retry succeeds. Two wasted calls, zero blocked progress — pure friction (Low), member of the actorPath-alias family (`E-actor-verbs-reject-actorpath-slot`) whose fix skipped these verbs, plus an unowned enumerate-in-error ask mirroring `E-set-niagara-param-error-omits-available-params`.
