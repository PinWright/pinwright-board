---
id: E-get-interaction-info-blueprint-bare
title: "interaction.get_interaction_info blueprintPath branch returns only {blueprintPath, blueprintName} — no component/trace/widget/events/chest read-back for a Blueprint target"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [interaction, get-interaction-info, readback, round-trip, blueprint]
encounters: 1
lastSeen: 2026-07-02T04:48:38+03:00
---

# `interaction.get_interaction_info` under-reports for a Blueprint target

`interaction.get_interaction_info` is documented "Get info about an interactable
Blueprint or actor", but when called with a **`blueprintPath`** it returns only
`{blueprintPath, blueprintName}` — it reports **none** of the state a caller
would want to confirm after authoring an interactable Blueprint:
(a) the interaction sphere/component, (b) the trace type/distance/radius,
(c) the prompt widget (`showOnHover`, prompt text), (d) the event dispatchers,
(e) the chest gameplay props (`bIsLocked`/`LidOpenAngle`/`OpenTime`/`LootTable`).
The full read-back exists **only** in the `actorName` branch (via the actor tags
added by `B-interaction-on-actor-configure-echo-noop`); the `blueprintPath`
branch was left emitting just name+path. So the one obvious "did my authoring
land?" RPC verifies nothing for the Blueprint that the sibling
`interaction.create_*`/`configure_*` verbs just wrote to.

## Process friction (why this is a struggle)

This is the load-bearing readback for the whole interaction-authoring flow, and
its uselessness for Blueprints is what forced the source-dive in the task that
surfaced it. After authoring everything correctly, the agent called
`get_interaction_info({blueprintPath:"/Game/Interactables/BP_TreasureChest"})`
and got back only `{"blueprintPath":"/Game/Interactables/BP_TreasureChest",
"blueprintName":"BP_TreasureChest"}`. Having no MCP-native way to confirm the
config, it fell back to (1) a last-resort read of the plugin C++
(`InteractionHandler.cpp` ~L917-1065, where the detailed emit lives solely in the
actor-tag branch) and (2) a generic `blueprint.inspect` dump to prove the
authoring actually landed (it did: `BPI_Interactable_C` interface, `InteractionSphere`
component, trace/widget/chest vars, 4 dispatchers). Both fallbacks were only
needed because the namespace-native verification RPC is bare for Blueprints —
this same gap is what made the co-filed `B-configure-chest-properties-no-cdo-writeback`
impossible to catch without reading source.

**Fix:** the `blueprintPath` branch should report the same (a)-(e) fields it
reports for an actor, read from the Blueprint's member vars / component template
/ CDO (the state `blueprint.inspect` already surfaces, just namespaced to the
interaction shape). Related — `B-interaction-on-actor-configure-echo-noop`
(IN-REVIEW) fixed the **actor** branch's readback via tags but did not touch the
**Blueprint** branch; this ticket covers that remaining branch.

severity rationale: impact=readback omits every field for a Blueprint target and forces a source-dive + generic-inspect fallback (soft blocker, workaround exists) × reach=interaction namespace, not every-session (no bump) -> Medium.

## History
- `#2-fix` `IN-REVIEW` developer — Fixed. `get_interaction_info`'s `blueprintPath` branch now reads the authored interaction config back off the compiled Blueprint instead of emitting only `{blueprintPath, blueprintName}`. Added `AddBlueprintInteractionReadback(Result, Blueprint)` in `Plugins/PinWright/Source/PinWright/Private/Handlers/Interaction/InteractionHandler.cpp` (anon-namespace helper after `CompileAndGetDefaultObject`, called from the blueprintPath branch at the `if (Blueprint)` block): (a) `component` = the SCS interaction-sphere node's name/class/`sphereRadius`; (b) `trace` = `TraceDistance`/`TraceType`; (c) `widget` = `bShowOnHover`/`bShowPromptText`/`PromptTextFormat`/`InteractionWidgetClass`; (d) `events` = the four `OnInteraction*` multicast dispatchers present on the GeneratedClass; (e) `chest` = `bIsLocked`/`LidOpenAngle`/`OpenAngle`/`OpenTime`/`LootTable`. Reuses the file's existing `EchoCompiledDefault` (FindFProperty + `ExportPropertyToJsonValue` off `GeneratedClass->GetDefaultObject()`) and the SCS-iteration pattern from `configure_interaction_trace`; every field is emitted only when its source property/component exists, so a bare Blueprint reports nothing extra. Distinct from `B-interaction-on-actor-configure-echo-noop` which fixed only the actor branch. Regression test `PinWright.interaction.get_interaction_info.BlueprintPathReportsConfig` in `Plugins/PinWright/Source/PinWright/Private/Tests/Gameplay/TestInteractionHandlers.cpp` authors a fresh in-code Actor Blueprint (create_interaction_component + configure_interaction_trace + configure_chest_properties + add_interaction_events via the real handlers) and asserts the blueprintPath readback reports the component/trace/chest sections and all four events (bare-branch revert fails it). Plugin builds clean (`Result: Succeeded`).
- `#1-initial-audit` `OPEN` reporter — Struggle-audit. After correct authoring, `get_interaction_info({blueprintPath:"/Game/Interactables/BP_TreasureChest"})` returned ONLY `{blueprintPath, blueprintName}` — none of sphere/trace/widget/dispatchers/chest props. Agent had to read `InteractionHandler.cpp` (~L917-1065; detailed emit lives only in the actorName/tag branch) and run a generic `blueprint.inspect` to prove the authoring landed. Distinct from `B-interaction-on-actor-configure-echo-noop` (IN-REVIEW): that fixed the ACTOR branch's readback via tags; the blueprintPath branch is still bare. This under-reporting is also why the co-filed `B-configure-chest-properties-no-cdo-writeback` needed a source-dive to confirm. Proposed: emit the (a)-(e) fields for a Blueprint path from member vars/component template/CDO.
