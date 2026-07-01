---
id: E-property-route-no-component-path-discovery
title: "property.* overlay never points the reflection route at actor.get_components — agents guess Component0 subobject names"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [property, component-path, discovery, docs]
---

# The generic `property.*` route has no component-path discovery step — `property.list` on an actor never surfaces its component subobjects

When an agent works an actor purely through the generic `property.*` reflection
route (the explicit intent: "use the generic `property.*` reflection route
throughout, not typed light/environment setters"), the fields it wants almost
always live on a **component**, not the actor itself — e.g. a directional
light's `Intensity` / `LightColor` / `Temperature` / `bUseTemperature` live on
the light component, fog density on the fog component, sky-light intensity on the
sky-light component. But:

- `property.list` on the actor (`DirectionalLight_0`) lists the **actor's** own
  editable fields and does not enumerate the actor's component subobject
  **paths**, so it is not actually a discovery step for "find the light
  component path" the way the task assumed.
- The only way to address the component through `property.*` is the full
  level-style subobject path, whose **leaf segment is a name the agent must
  already know** (`LightComponent0`, `ExponentialHeightFogComponent0`,
  `SkyLightComponent0`) — and those default-subobject names are not the obvious
  guess (`DirectionalLightComponent`, `Component`, `LightComponent` all fail).

So the agent is forced to guess the component subobject name. In this task that
cost a string of dead calls before the right name was found, repeated per actor:

- `property.list DirectionalLight_0.DirectionalLightComponent` → `OBJECT_NOT_FOUND`
- `property.list …DirectionalLightComponent nameMatch=temperature` → `OBJECT_NOT_FOUND`
- `property.list …DirectionalLightComponent nameMatch=intensity` → `OBJECT_NOT_FOUND`
- `property.list …DirectionalLightComponent0` (wrong subobj) → silently fell to the World
- … then finally `LightComponent0` (the real default-subobject name) worked.
- Same dance again on `ExponentialHeightFog_0.Component` (`OBJECT_NOT_FOUND`) and
  `SkyLight_0.LightComponent` (`OBJECT_NOT_FOUND`) before the real
  `ExponentialHeightFogComponent0` / `SkyLightComponent0` names were used.

Friction note (verbatim): *"actor.component dotted paths (e.g.
DirectionalLight_0.DirectionalLightComponent) do not resolve for property.list
and one wrong subobject-name guess (ExponentialHeightFogComponent0) silently fell
back to the World; worked around by using property.get's nested-dot route and the
real subobject names (LightComponent0, SkyLightComponent0) discovered from the
level path."*

This is the **process / discoverability** layer of the same workflow whose
silent-success *tool bug* the judge already filed as
[`B-property-path-silent-world-fallback`](B-property-path-silent-world-fallback.md):
that ticket fixes the wrong-object fallback so a bad guess *errors* instead of
returning the World. This ticket is the orthogonal ergonomic gap that makes the
agent guess in the first place — even with the resolver fixed, the agent still
has no affordance in the `property.*` route to *learn* the component subobject
name, so it would still burn N failed `property.list` calls converging on it.

The plugin already owns the right discovery + component-aware verbs — they are
simply never surfaced from the `property.*` route or its overlay:

- `actor.get_components` lists every component (class + **name**) on an actor —
  exactly the subobject names needed for the `property.*` path leaf.
- `actor.get_component_property` / `actor.set_component_properties` take an actor
  + a **named component**, sidestepping the full level-style subobject path
  entirely (no `…:PersistentLevel.Actor.Component0` construction, no World-fallback
  trap).

## Fix (docs-only)

**Scope narrowed to a docs steer.** The earlier draft also floated a code option
(emit a `components:[{name,classPath,objectPath}]` block from `property.list` when
given an actor path). That is dropped: it would bolt component-enumeration onto the
load-bearing shared reflection-list handler purely to duplicate what
`actor.get_components` already returns in one call (name + class + full subobject
`path`, ComponentHandler.cpp:371-375) — a band-aid for a one-call saving. The
friction is a discoverability/docs gap, not a code gap, so the proportionate fix is
docs-only.

In `Docs/wiki-src/property.md`, add an explicit "properties on a component" note to
the `property.list` / `property.set` overlay H3 sections (so it surfaces when an
agent calls `call("property.list")` / `call("property.set")`): a property that lives
on a component (light Intensity/Color/Temperature, fog FogDensity, sky-light
Intensity) is not reachable from the bare actor path — first call
`actor.get_components(<actorName>)` to read the component **name** (UE default
subobjects are named `…Component0`, not `…Component`/`…Component`), then target
`<actorPath>:PersistentLevel.<Actor>.<Component0>` — or skip the path construction
entirely and use the typed `actor.get_component_property` /
`actor.set_component_properties` verbs (actor + named component, no level-style path,
no World-fallback trap). Also fix the overlay's only `actor.get_components` mention,
which currently steers *away* from it ("prefer actor.describe over stitching together
property.list, actor.get_components…") — it must instead acknowledge that an agent
committed to the `property.*` route uses `actor.get_components` to find the component
subobject name.

**Workaround:** before any component-level `property.*` call, run
`actor.get_components(<actorName>)` to read the real component subobject name
(`LightComponent0`, `ExponentialHeightFogComponent0`, `SkyLightComponent0`), or
use `actor.get_component_property` / `actor.set_component_properties` with the
component **name** instead of building a level-style subobject path.

## History
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of a "relight via generic property.* reflection" task (distinct PROCESS angle from the judge's tool-bug ticket `B-property-path-silent-world-fallback`). The task's properties lived on light/fog/sky components, but `property.list` on the actor surfaces no component subobject paths, so the agent guessed the default-subobject name and burned ~3 `OBJECT_NOT_FOUND` `property.list` calls on `DirectionalLight_0.DirectionalLightComponent` (+ a wrong `…Component0` guess that silently fell to the World) before `LightComponent0` worked — then repeated the dead-end guesses on `ExponentialHeightFog_0.Component` and `SkyLight_0.LightComponent` before `ExponentialHeightFogComponent0` / `SkyLightComponent0`. Friction note (verbatim) quoted in body. Proposed: have `property.list` on an actor emit its component subobject names/paths, and/or amend `docs/wiki-src/property.md` (tag `docs`) to point the property.* route at `actor.get_components` for the component name + the typed `actor.get_component_property` / `actor.set_component_properties` verbs. Dedup: ripgrep + qmd (pds) across OPEN/closed — `B-property-path-silent-world-fallback` is the silent-World *resolver* bug (not the discovery gap); `E-spawn-returns-actor-not-component-path` is scoped to typed *spawn* verbs withholding componentPath (this task spawned nothing); `E-component-read-filter` (DONE) filters an already-known component tree; none cover the property.* route lacking a component-name discovery step.
- `#2-evidence-blueprint-cdo` `OPEN` reporter — Cross-task evidence: same discoverability gap on a Blueprint generated-class **CDO** (not a level actor). Task `property.reset` on `/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic` (CDO of `BP_Light_Bulb_Basic_C`): the story told the agent to call `property.list editableOnly:true` and pick a light-intensity/brightness/color/attenuation scalar to tweak, but the bulb's light scalars live on a nested `PointLightComponent`, not the CDO root. `property.list editableOnly:true` returned only **30 inherited AActor engine fields** — no bulb-specific or light-component scalar was reachable from the CDO path — so the agent could not find the intended intensity property and fell back to tweaking the inherited `AActor` `SpriteScale` instead (which then exposed the separate CDO-reset bug the judge filed as `B-property-reset-cdo-uses-self-not-archetype`). Friction note (verbatim): *"editableOnly:true on this bulb returned only 30 inherited AActor engine fields (no bulb-specific or light-component scalars at the CDO root, since intensity lives on a nested PointLightComponent)"*. Confirms the proposed fix should cover Blueprint **CDO** targets too (not just placed level actors): `property.list` on a CDO/actor should surface its component subobject names so the next call can target the light component's `Intensity`/`LightColor`. The oversized-spill side-symptom (this property.list response spilled to a HttpResponses file requiring a Read) is the working-as-designed mechanism already covered by `E-http-response-spill` (DONE) / `F-rpc-property-omit-oversized-opt-in` (DONE) — not re-filed.
- `#3-reword-docs-only` `IN-REVIEW` developer — Reworded to docs-only scope per adversarial review and implemented. Three validity lenses agreed the gap is real, code-verified, and non-duplicate (property.list at UtilityPropertyHandler.cpp:1592 iterates only `RootObject->GetClass()` and emits no component block; `actor.get_components` ComponentHandler.cpp:322 already returns each component's name+class+full subobject `path` in one call; the overlay's only `actor.get_components` mention at property.md:9 steered *away* from it). Dropped the code option (#1, emit a `components` block from property.list) as mis-scoped — it would duplicate `actor.get_components` inside the load-bearing shared reflection handler for a one-call saving. Implemented option #2 only: in `Docs/wiki-src/property.md` added a "Properties that live on a component" note to the `### property.list` and `### property.set` overlay H3 sections (call `actor.get_components` for the real `…Component0` subobject name, then target the level-style path, or skip it via the typed `actor.get_component_property` / `actor.set_component_properties` verbs), and rewrote the line-9 cross-cluster note so the property.* route is pointed *at* `actor.get_components` instead of away from it. Regression test added in `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` (`FWikiHandlerPropertyComponentDiscoveryTest`) renders the `property.list` and `property.set` method pages via `WikiHandler::RenderPage` (production overlay loader) and asserts each names `actor.get_components` and the `Component0` default-subobject convention — fails iff the H3 overlay notes are reverted. Files: `Docs/wiki-src/property.md`, `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp`. Not compiled/tested here (later phase).
