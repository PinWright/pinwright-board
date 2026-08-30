---
id: F-sky-cloud-reflection-actors
title: "First-class SkyAtmosphere / VolumetricCloud / ReflectionCapture spawn"
status: DONE
severity: Medium
category: feature
tags: [environment, lighting, sky, atmospherics, reflection-capture]
---

# First-class SkyAtmosphere / VolumetricCloud / ReflectionCapture spawn

The plugin's environment / lighting surface lacks first-class spawn RPCs for
the modern UE volumetric atmospherics and reflection actors. `environment.build.create_sky_sphere`
in `EnvironmentHandler.cpp` still spawns the legacy `BP_Sky_Sphere` actor (not
shipped with bare Unreal projects), and `lighting.spawn_light` / `lighting.spawn_sky_light`
in `LightingHandler.cpp` only cover `ALight` subclasses + `ASkyLight`. Grep across
`docs/rpc-method-reference.generated.md` and `Source/PinWright/Private/Handlers/`
finds zero references to `ASkyAtmosphere`, `AVolumetricCloud`, or
`A*ReflectionCapture`. Authoring agents that want a modern outdoor lighting rig
have to fall back to `actor.spawn` by class path and then chase down
component-on-actor `property.set` calls without a typed verb.

Add three sibling RPCs under the `environment` namespace, matching the existing
`lighting.spawn_*` pattern (typed location/rotation/name + a small set of the
most-tuned UPROPERTYs, with the rest reachable through `property.set` on the
returned actor path):

- `environment.spawn_sky_atmosphere` — spawns `ASkyAtmosphere`. Exposed
  UPROPERTYs (3–5 most-tuned): `RayleighScattering` (FLinearColor),
  `RayleighScatteringScale` (float), `MieScatteringScale` (float),
  `MieAbsorptionScale` (float), `MultiScatteringFactor` (float).
- `environment.spawn_volumetric_cloud` — spawns `AVolumetricCloud`. Exposed
  UPROPERTYs: `LayerBottomAltitude` (float, km), `LayerHeight` (float, km),
  `TracingStartMaxDistance` (float, km), `TracingMaxDistance` (float, km).
  `Material` (soft path to `UMaterialInterface`) optional.
- `environment.spawn_reflection_capture(shape: "Sphere"|"Box", ...)` — spawns
  `ASphereReflectionCapture` or `ABoxReflectionCapture` based on `shape`.
  Exposed UPROPERTYs: `InfluenceRadius` (float, sphere only),
  `BoxTransitionDistance` (float, box only), `Brightness` (float),
  `ReflectionSourceType` (enum: CapturedScene|SpecifiedCubemap),
  `Cubemap` (soft path).

All three return `{ actorPath, actorLabel, className }` so callers can chain
`property.set` for the long tail. Place handlers in
`Private/Handlers/Environment/EnvironmentHandler.cpp` (atmosphere + cloud,
since they are world-environment) and a new
`Private/Handlers/Environment/ReflectionCaptureHandler.cpp` (capture actors
have meaningful per-shape divergence that warrants its own file).

**Why one ticket, not three:** these three actors form one coherent
"modern outdoor lighting" cluster in UE — they are documented together in the
Volumetric Cloud / Sky Atmosphere quick-start, share the same `actor.spawn` +
`property.set` workaround pain today, and each handler is ~30–50 lines. Splitting
would fragment a single small surface across three files with three nearly
identical `REGISTER_RPC_HANDLER` blocks.

**Workaround:** Call `actor.spawn` with `classPath="/Script/Engine.SkyAtmosphere"`
(etc.) and then `property.set` for each tuned field on the actor's root component.

**Fix:** The RPCs already exist; fix the live-editor spawn branch. Add a direct active-world spawn fallback after `SpawnActorInActiveWorld<T>`'s editor subsystem path returns null, matching the working `actor.spawn` direct-world behavior. Preserve chainable response fields by returning the live actor object path, actor label, and class short name.

## History
- `#1-initial-report` `OPEN` reporter — Modern atmospherics actors (ASkyAtmosphere, AVolumetricCloud, ASphereReflectionCapture, ABoxReflectionCapture) have no first-class spawn RPC; only legacy BP_Sky_Sphere via `environment.build.create_sky_sphere` and `ALight` / `ASkyLight` via `lighting.spawn_*`. Confirmed via grep across `rpc-method-reference.generated.md` and `Source/EditorAutomationRpcGateway/Private/Handlers/`: zero references to the four target classes. Add `environment.spawn_sky_atmosphere`, `environment.spawn_volumetric_cloud`, `environment.spawn_reflection_capture(shape)` exposing 3–5 most-tuned UPROPERTYs each with the rest reachable via `property.set`.
- `#2-typed-environment-spawn-rpcs` `IN-REVIEW` developer — Added environment.spawn_sky_atmosphere and environment.spawn_volumetric_cloud to EnvironmentHandler.cpp; new ReflectionCaptureHandler.cpp with environment.spawn_reflection_capture(shape). All use SpawnActorInActiveWorld<T> + FindComponentByClass + a PropertyUtils::ApplyJsonValueToProperty/FindPropertyCI loop (no new shared helper — three callers below the lift threshold). Wiki overlay updated; 5 regression tests appended to TestEnvironmentHandlers.cpp (per-handler round-trip + a method-found counterfactual).
- `#3-returned-spawn-failed-all-three` `OPEN` tester — Returned: all three new RPCs return `SPAWN_FAILED` against the live editor (L_Core map). Verified schemas via `environment.spawn_sky_atmosphere?`, `environment.spawn_volumetric_cloud?`, `environment.spawn_reflection_capture?` — all three register with correct params. Then invoked each with realistic args: `environment.spawn_sky_atmosphere` → `{"code":"SPAWN_FAILED","message":"Failed to spawn SkyAtmosphere actor"}`; `environment.spawn_volumetric_cloud` → `{"code":"SPAWN_FAILED","message":"Failed to spawn VolumetricCloud actor"}`; `environment.spawn_reflection_capture` (shape=Sphere) → `{"code":"SPAWN_FAILED","message":"Failed to spawn reflection capture actor"}`. As a control, the documented workaround `actor.spawn` with `classPath="/Script/Engine.SkyAtmosphere"` succeeded in the same editor session and spawned an actor (cleaned up after). Root cause is in `SpawnActorInActiveWorld<T>` returning nullptr for these classes — likely the interactive-editor `UEditorActorSubsystem::SpawnActorFromClass` path rejects these scene-actor types; `actor.spawn` (SpawnHandler.cpp) uses a different code path that works. Fix needs to route these handlers through the same spawn path that `actor.spawn` uses, or fall back to a direct `EditorWorld->SpawnActor` when the subsystem returns null.
- `#4-live-editor-spawn-fallback` `IN-REVIEW` developer — Reformulated the returned scope around the already-added RPCs: fixed the interactive editor spawn branch by falling back to direct active-world spawning when UEditorActorSubsystem returns null, preserved chainable actorPath/actorLabel/className response fields, and extended environment spawn tests for response shape.
- `#5-verify-live-spawn` `DONE` tester — Verified: all three RPCs spawned successfully against L_Core. `environment.spawn_sky_atmosphere {name:"McpVerifySkyAtmosphere"}` → `success:true`, actorClass `SkyAtmosphere`, returned actorPath/actorLabel/className. `environment.spawn_volumetric_cloud {name:"McpVerifyVolumetricCloud"}` → `success:true`, actorClass `VolumetricCloud`. `environment.spawn_reflection_capture {shape:"Sphere",name:"McpVerifyReflectionCapture"}` → `success:true`, actorClass `SphereReflectionCapture`. No more SPAWN_FAILED. All three test actors cleaned up via `actor.delete` batch (deletedCount=3).
- `#6-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
