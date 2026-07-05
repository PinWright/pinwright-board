---
id: E-get-interaction-info-blueprint-bare
title: "interaction.get_interaction_info blueprintPath branch returns only {blueprintPath, blueprintName} — no component/trace/widget/events/chest read-back for a Blueprint target"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [interaction, get-interaction-info, readback, round-trip, blueprint]
encounters: 2
lastSeen: 2026-07-05T11:06:33+03:00
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

## Follow-up: the #2-fix readback still omits the switch gameplay props

The #2-fix `AddBlueprintInteractionReadback` covers the chest/door gameplay-prop
configurators — its `(e)` section echoes `bIsLocked`/`LidOpenAngle`/`OpenAngle`/`OpenTime`/`LootTable`
(from `configure_chest_properties` / `configure_door_properties`) — but it has **no
section for the third sibling gameplay-prop configurator, `configure_switch_properties`**.
That verb bakes `SwitchType`/`bCanToggle`/`ResetTime` (and `bIsActivated`) onto the
switch CDO (InteractionHandler.cpp L750-753), yet `get_interaction_info` surfaces none
of them on **either** target: the Blueprint branch's `AddBlueprintInteractionReadback`
(L296-304, the `(e)` chest/door block) has no switch echo, and the actor branch
(L1015-1050) reads back only widget/trace/component tags. So after building a switch
interactable, the one namespace-native "confirm my switch" RPC still verifies nothing
switch-specific, and the caller must fall back to a generic `blueprint.inspect` CDO dump.

Fix scope: extend `AddBlueprintInteractionReadback`'s gameplay-prop block with a
`switch` section echoing `SwitchType`/`bCanToggle`/`ResetTime`/`bIsActivated` (mirroring
the existing `EchoCompiledDefault` chest/door pattern), so the readback covers all three
`configure_*_properties` siblings — not just chest/door.

## History
- `#3-additional-switch-props` `IN-REVIEW` reporter — Additional evidence (new angle: the fix is incomplete for the switch sibling). The #2-fix landed and is live at HEAD — the `blueprintPath` branch is no longer bare — but its `AddBlueprintInteractionReadback` `(e)` block covers only the chest/door gameplay-prop configurators and omits the third sibling, `configure_switch_properties`. Repro: `blueprint.create` BP_PuzzleLever (Actor) → `create_interaction_component` → `configure_interaction_widget` → `configure_switch_properties({switchType:"Lever", canToggle:true, resetTime:3})` → compile+save → `get_interaction_info({blueprintPath:"/Game/Puzzle/BP_PuzzleLever"})` returns `{blueprintPath, blueprintName, component:{componentName:"InteractionSphere", componentClass:"SphereComponent", sphereRadius:150}, widget:{showOnHover:true, showPromptText:true, promptTextFormat:"Press E to Flip", widgetClass:null}, events:[OnInteractionStart, OnInteractionEnd, OnInteractableFound, OnInteractableLost]}` — **no `switch` section, no switchType/canToggle/resetTime**. The switch config DID persist: `blueprint.inspect({includeProperties:true})` shows the CDO carries `bCanToggle:true`, `ResetTime:3`, `SwitchType:Lever`. So the switch props round-trip through the generic inspector but not through the namespace-native readback — the same "readback omits a field, workaround via blueprint.inspect exists" struggle this ticket is about, just for the switch field-set the fix didn't add. Culprit method `interaction.get_interaction_info`; source line `AddBlueprintInteractionReadback` §(e) at InteractionHandler.cpp L296-304 (no switch echo). Bumped encounters 1 -> 2.
- `#2-fix` `IN-REVIEW` developer — Fixed. `get_interaction_info`'s `blueprintPath` branch now reads the authored interaction config back off the compiled Blueprint instead of emitting only `{blueprintPath, blueprintName}`. Added `AddBlueprintInteractionReadback(Result, Blueprint)` in `Plugins/PinWright/Source/PinWright/Private/Handlers/Interaction/InteractionHandler.cpp` (anon-namespace helper after `CompileAndGetDefaultObject`, called from the blueprintPath branch at the `if (Blueprint)` block): (a) `component` = the SCS interaction-sphere node's name/class/`sphereRadius`; (b) `trace` = `TraceDistance`/`TraceType`; (c) `widget` = `bShowOnHover`/`bShowPromptText`/`PromptTextFormat`/`InteractionWidgetClass`; (d) `events` = the four `OnInteraction*` multicast dispatchers present on the GeneratedClass; (e) `chest` = `bIsLocked`/`LidOpenAngle`/`OpenAngle`/`OpenTime`/`LootTable`. Reuses the file's existing `EchoCompiledDefault` (FindFProperty + `ExportPropertyToJsonValue` off `GeneratedClass->GetDefaultObject()`) and the SCS-iteration pattern from `configure_interaction_trace`; every field is emitted only when its source property/component exists, so a bare Blueprint reports nothing extra. Distinct from `B-interaction-on-actor-configure-echo-noop` which fixed only the actor branch. Regression test `PinWright.interaction.get_interaction_info.BlueprintPathReportsConfig` in `Plugins/PinWright/Source/PinWright/Private/Tests/Gameplay/TestInteractionHandlers.cpp` authors a fresh in-code Actor Blueprint (create_interaction_component + configure_interaction_trace + configure_chest_properties + add_interaction_events via the real handlers) and asserts the blueprintPath readback reports the component/trace/chest sections and all four events (bare-branch revert fails it). Plugin builds clean (`Result: Succeeded`).
- `#1-initial-audit` `OPEN` reporter — Struggle-audit. After correct authoring, `get_interaction_info({blueprintPath:"/Game/Interactables/BP_TreasureChest"})` returned ONLY `{blueprintPath, blueprintName}` — none of sphere/trace/widget/dispatchers/chest props. Agent had to read `InteractionHandler.cpp` (~L917-1065; detailed emit lives only in the actorName/tag branch) and run a generic `blueprint.inspect` to prove the authoring landed. Distinct from `B-interaction-on-actor-configure-echo-noop` (IN-REVIEW): that fixed the ACTOR branch's readback via tags; the blueprintPath branch is still bare. This under-reporting is also why the co-filed `B-configure-chest-properties-no-cdo-writeback` needed a source-dive to confirm. Proposed: emit the (a)-(e) fields for a Blueprint path from member vars/component template/CDO.
