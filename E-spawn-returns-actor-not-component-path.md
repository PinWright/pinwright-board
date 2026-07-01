---
id: E-spawn-returns-actor-not-component-path
title: "Typed environment spawn verbs return only actorPath, not the componentPath they applied properties to — property readback needs a get_components round-trip"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [environment, spawn, sky-atmosphere, property-readback, component-path, docs]
---

# Typed spawn verbs return the actor path but apply properties to the component — readback forces a get_components round-trip

`environment.spawn_sky_atmosphere` (and its siblings `spawn_volumetric_cloud` /
`spawn_reflection_capture`) fan out the optional `properties` bag to the spawned
actor's **primary component** (`USkyAtmosphereComponent`,
`UVolumetricCloudComponent`, `UReflectionCaptureComponent`), but the response
returns only `{ actorPath, actorLabel, className }` plus actor verification
fields — it never returns the **component** object path it actually wrote to.

The two facts collide on any follow-up readback or property tweak:

- The property the spawn just set (`RayleighScatteringScale`) lives on the
  component, not the actor.
- The only object path the caller gets back is `actorPath` — the actor.

So a caller who wants to verify the applied value has no component path to hand
to `property.get`. The wiki overlay
(`docs/wiki-src/environment.md`) actively steers them wrong here:

> All three return `{ actorPath, actorLabel, className }` … `actorPath` is the
> spawned actor's object path, so callers can chain `call("property.set")` or
> `call("property.get")` for the long tail of UPROPERTYs not in the most-tuned
> list.

Chaining `property.get` from `actorPath` for a **component** UPROPERTY is exactly
what fails: in this task the agent did that and got
`[PROPERTY_NOT_FOUND] Property RayleighScatteringScale not found on object
/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome` (the property is on the
`SkyAtmosphereComponent`, not the actor/level object). Recovery cost two extra
calls: `actor.get_components(MainSkyAtmosphere, class=SkyAtmosphereComponent)` to
discover the real component path, then a second `property.get` on that path.

This is a discoverability/ergonomic gap, not a tool bug — the spawn succeeded and
the property *was* applied (final readback confirmed `0.04`). The friction is the
three-call dance (`property.get` wrong → `get_components` → `property.get` right)
to read back a value the spawn itself had already set, caused by the response
withholding the one path the caller needs.

**Fix (ergonomic, additive):** have the three typed spawn verbs also return the
object path of the component they fanned `properties` onto — e.g. a
`componentPath` (and/or `componentName` / `componentClass`) field alongside
`actorPath` — so callers can `property.get` / `property.set` the component's long
tail directly, no `get_components` discovery hop. The spawn handler already has
the component pointer in hand at apply time, so emitting its path is cheap. The
`rejected` array already tells callers *which* keys missed; a `componentPath`
tells them *where* the accepted ones landed.

**Docs (`docs/wiki-src/environment.md`, tagged `docs`):** correct the "chain
`property.get` from the returned `actorPath`" guidance — for the
component-level UPROPERTYs these verbs tune (Rayleigh/Mie scattering, cloud layer
altitudes, reflection influence radius), `property.get` on `actorPath` returns
`PROPERTY_NOT_FOUND`. Point callers at the new `componentPath` (once added), or in
the interim document that the component path must be discovered via
`actor.get_components` before a component-property readback.

**Workaround:** after spawning, call `actor.get_components(<name>,
componentClass=<…Component>)` to get the component object path, then
`property.get` / `property.set` against that path rather than `actorPath`.

## History
- `#2-emit-componentpath` `IN-REVIEW` developer — Implemented the additive fix (code + docs) as scoped. Code: the three typed spawn verbs now emit `{ componentPath, componentName, componentClass }` for the component they fanned `properties` onto, alongside the existing actor fields. Added a shared helper `AddSpawnedComponentFields(Response, UActorComponent*)` in `Source/.../Handlers/Environment/EnvironmentSpawnHelpers.h` (emits `Comp->GetPathName()` / `GetName()` / `GetClass()->GetName()`), wired into `EnvironmentHandler.cpp` (`spawn_sky_atmosphere`, `spawn_volumetric_cloud`) and `ReflectionCaptureHandler.cpp` (`spawn_reflection_capture`). Decision note on the adversarial reword-to-docs-only objection: although nested dotted `property.get(actorPath, "SkyAtmosphereComponent.RayleighScatteringScale")` and the typed `actor.get_component_property` both resolve in one call, BOTH require the caller to already know the reflected component-property name (the discovery gap the reporter actually hit), so the additive componentPath is the proportionate fix — not gold-plating. This is materially distinct from the docs-only narrowing of sibling `E-property-route-no-component-path-discovery`, whose dropped code option would have bolted enumeration onto the *load-bearing shared* `property.list` handler to duplicate `actor.get_components`; here the change adds one field to three narrow dedicated spawn handlers that already hold the component pointer (zero new lookups, no shared-handler risk, no duplication — no verb currently returns this path). Consistent with the OPEN audio sibling's option #1 ("return a resolvable componentPath") and the board's component-handle theme. Docs: corrected `Docs/wiki-src/environment.md` — the "chain property.get from actorPath" guidance now steers callers to chain from the new `componentPath` (with an explicit note that bare component knobs on `actorPath` return PROPERTY_NOT_FOUND), and the See-also property.md line updated likewise. Test: extended the existing round-trip tests in `Source/EditorAutomationRpcGateway/Private/Tests/World/TestEnvironmentHandlers.cpp` — the shared `InvokeEnvironmentSpawnAndValidateResponse` helper now asserts `componentPath`/`componentName`/`componentClass` are present, `componentPath` is non-empty and distinct from `actorPath`, and each per-verb test (sky/cloud/sphere/box) asserts `componentPath` equals the live typed component's `GetPathName()`. Fails iff the componentPath emit is reverted. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentSpawnHelpers.h`, `.../EnvironmentHandler.cpp`, `.../ReflectionCaptureHandler.cpp`, `Docs/wiki-src/environment.md`, `Source/EditorAutomationRpcGateway/Private/Tests/World/TestEnvironmentHandlers.cpp`. Not compiled/tested here (later phase).
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of an `environment.build` outdoor-lighting task (distinct PROCESS angle from the judge's tool-bug ticket `B-create-sky-sphere-stale-path`). `environment.spawn_sky_atmosphere {name:MainSkyAtmosphere, Rayleigh=0.04, Mie=0.05}` succeeded, but verifying the applied value cost a 3-call dance: `property.get` on the spawned object guessed the actor/map path and returned `[PROPERTY_NOT_FOUND] Property RayleighScatteringScale not found on object /Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome` (the property is on the SkyAtmosphereComponent), then `actor.get_components(MainSkyAtmosphere, class=SkyAtmosphereComponent)` discovered the real component path, then a second `property.get` read back 0.04. Friction note (verbatim): "first property.get guessed the wrong component subobject name and needed actor.get_components to find the real path." Root: the typed spawn verbs apply `properties` to the actor's primary component (per `docs/wiki-src/environment.md`) but the response returns only `{actorPath, actorLabel, className}` — never the componentPath written to — and the overlay's "chain property.get from actorPath" advice misleads straight into PROPERTY_NOT_FOUND for component UPROPERTYs. Proposed: emit `componentPath` (+ name/class) from the spawn response and fix the overlay's readback guidance. Dedup: ripgrep + qmd across OPEN/closed found no existing ticket — `E-component-read-filter` (DONE) scopes `get_components` reads but doesn't make spawn return the component path; `B-get-components-renders-empty-to-caller` (DONE) is the empty-render transport bug; `B-inspect-object-omits-component-properties` is a different read verb.
