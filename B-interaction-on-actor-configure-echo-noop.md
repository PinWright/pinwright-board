---
id: B-interaction-on-actor-configure-echo-noop
title: "interaction.configure_interaction_widget_on_actor reports success but writes nothing to the actor (echo-only no-op); get_interaction_info can't read it back"
status: IN-REVIEW
severity: High
category: bug
tags: [interaction, on-actor, no-op, round-trip, readback]
---

# interaction.configure_interaction_widget_on_actor reports success but writes nothing to the actor (echo-only no-op)

`interaction.configure_interaction_widget_on_actor` returns a success
response that echoes the supplied `widgetText` / `showOnHover` / `offsetZ`
(and a `widgetClass` field), strongly implying the widget config was applied
to the actor. It is **not**: the handler never touches the actor — it only
constructs a JSON response from the input args and sends it. The config does
not persist and does not round-trip. A caller doing the natural "configure
then read back to confirm" loop is misled: the configure call looks like it
worked, but `interaction.get_interaction_info` reports nothing was stored.

This is **silent success-with-no-effect**: a fully successful (`ok:true`,
`isError:false`) response that echoes the requested state while applying none
of it.

The same echo-only pattern affects the sibling on-actor configurators:
- `interaction.configure_interaction_trace_on_actor` — echoes
  `traceDistance` / `traceChannel` / `useComplexCollision`, writes nothing.
- `interaction.create_interaction_component_on_actor` — this one *does* spawn
  a bare `USceneComponent` named `InteractionComponent` and registers it, but
  it discards the `interactionDistance` / `requiresLineOfSight` it echoes back
  (a `USceneComponent` has no such properties), so those values are also lost.

The readback side, `interaction.get_interaction_info`, makes the gap visible:
for the actor branch it only emits `actorName` + `actorClass` and never reports
any component / trace / widget state, so even if config *were* applied there is
no way to confirm it.

## Repro (verbatim, replayed live against mcp__editor-automation__call)

Target actor: `StaticMeshActor_1` (label "UELogo", a `StaticMeshActor` in the
open `ExampleProjectWelcome` level — confirmed via `actor.find_by_name`).

1. `interaction.configure_interaction_widget_on_actor`
   args: `{"actorName":"StaticMeshActor_1","widgetText":"Press E to Open","showOnHover":true,"offsetZ":90}`
   → `{"actorName":"StaticMeshActor_1","widgetClass":"","widgetText":"Press E to Open","showOnHover":true,"offsetZ":90}`
   (success; echoes the inputs — looks applied)

2. `interaction.get_interaction_info`
   args: `{"actorName":"StaticMeshActor_1"}`
   → `{"actorName":"StaticMeshActor_1","actorClass":"StaticMeshActor"}`
   (no `widgetText` / `offsetZ` / `showOnHover` / widget state — nothing
   round-tripped)

## Source confirmation

`Source/EditorAutomationRpcGateway/Private/Handlers/Interaction/InteractionHandler.cpp`:
- `configure_interaction_widget_on_actor` (~L1030-1059): reads the args, builds
  `Resp` from them, `Ctx.SendSuccess(Resp)` — never resolves or mutates the
  actor at all.
- `configure_interaction_trace_on_actor` (~L1002-1028): same echo-only shape;
  doesn't even look up the actor.
- `create_interaction_component_on_actor` (~L943-1000): spawns/registers a bare
  `USceneComponent` but stores none of the echoed distance / line-of-sight.
- `get_interaction_info` (~L917-933): actor branch sets only `actorName` +
  `actorClass`.

**Fix (proposed):** either (a) make the on-actor configurators actually persist
their settings onto a real interaction component on the actor and have
`get_interaction_info` read that state back (true round-trip), or (b) if the
plugin cannot model interaction state on an arbitrary actor, have these handlers
return a clean error instead of a success that echoes un-applied input. A
success response that mirrors the request but changes nothing is the worst case
— it actively misleads the configure-then-verify workflow.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live: `configure_interaction_widget_on_actor` on `StaticMeshActor_1` returned success echoing `widgetText:"Press E to Open"`, `showOnHover:true`, `offsetZ:90`; the immediate `get_interaction_info` readback returned only `{"actorName":"StaticMeshActor_1","actorClass":"StaticMeshActor"}` — none of the widget state persisted. Source (`InteractionHandler.cpp`) confirms the on-actor configure handlers build the response from inputs and never write to the actor (silent success-with-no-effect), and the `get_interaction_info` actor branch emits only name+class.
- `#2-fix` `IN-REVIEW` developer — Implemented option (a), true round-trip persistence. The three on-actor configurators now persist their settings onto the resolved world actor as namespaced `AActor::Tags` (`EAMcp.Interaction.<Field>=<value>`) — a real, serialized property on any arbitrary actor — and `interaction.get_interaction_info`'s actor branch reads them back into `widget`/`trace`/`component` sub-objects. Also fixed the related sub-bug where `configure_interaction_widget_on_actor`/`configure_interaction_trace_on_actor` "succeeded" on a nonexistent actor: both now resolve the actor first and return `ACTOR_NOT_FOUND` (matching `create_interaction_component_on_actor`'s existing behavior) and `NO_WORLD` if there is no editor world. `create_interaction_component_on_actor` additionally persists the previously-dropped `interactionDistance`/`requiresLineOfSight` as tags. File: `Source/EditorAutomationRpcGateway/Private/Handlers/Interaction/InteractionHandler.cpp` (added `SetInteractionTag`/`GetInteractionTag` helpers; updated all four handlers). Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestInteractionHandlers.cpp` → `EditorAutomationRpcGateway.interaction.on_actor_config.RoundTripsThroughGetInfo` spawns an actor, configures widget+trace+component on it, asserts every value round-trips through `get_interaction_info`, and asserts the configurators now error (`ACTOR_NOT_FOUND`) on a missing actor — it would fail against the old echo-only handlers (empty readback) and the old silent-success-on-missing-actor behavior.
