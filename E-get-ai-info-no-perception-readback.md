---
id: E-get-ai-info-no-perception-readback
title: "`ai.get_ai_info` returns only `{controllerClass}` — it can't read back the perception component / SensesConfig the `ai` write verbs configure, so the configure→verify round-trip is structurally unverifiable"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [ai, perception, readback, get_ai_info, round-trip, verify, discovery, docs]
---

# The `ai` namespace's own read-back verb can't confirm the perception state the namespace's write verbs set

The natural "did it work?" read after wiring an AI Controller's perception is
`ai.get_ai_info` — it lives in the same namespace as the write verbs
(`add_ai_perception_component`, `configure_sight_config`,
`configure_hearing_config`, `configure_damage_sense_config`,
`set_perception_team`) and the story explicitly asks to "read the controller's
AI info back so I can confirm everything is in place." But `ai.get_ai_info`
returns only `{"controllerClass":"BP_GuardAIController_C"}` — it resolves the
controller (no `ASSET_NOT_FOUND`) yet reports **nothing** about the AI Perception
component, the `SensesConfig` array, or any sight/hearing/damage/team field that
the configure verbs just wrote. The read surface that the namespace appears to
advertise for verification is **structurally incapable** of confirming its own
namespace's writes.

This is a distinct PROCESS angle from the silent-no-op write bug filed by the
judge (`B-configure-sense-config-silent-noop`): that ticket's identity, root
cause, and fix are about `configure_*_config` echoing input without ever touching
`SensesConfig`. **Even once those writes are fixed, `ai.get_ai_info` still would
not surface the perception/sense state**, so the configure→readback round-trip
the story asks for would remain unverifiable through the `ai` surface alone. The
readback thinness is independently fixable and outlives the write bug.

This is the same shape as `E-game-framework-info-not-asset-readback`: a
namespace's `get_*_info` verb that cannot confirm the asset/component the
namespace's `set_*`/`configure_*` verbs mutate.

## Friction evidence (this task, `ai.configure_sight_config` guard-AI story, 16 calls)

Every write call succeeded and echoed its values (controller, perception,
sight{1500/1800/70}, hearing{1200}, damage, team 1), then the readback collapsed.
Quoting the friction note:

> "Readback gap surfaced: get_ai_info resolves the controller (no
> ASSET_NOT_FOUND) but omits perception/sight detail entirely; configure_sight_config
> only echoes the input rather than the persisted state, so persistence is
> unobservable — I retried get_ai_info with the full object path to rule out a
> transient empty payload, same thin result."

Call-log shape — the **trial-and-error retry** is the PROCESS cost:
- `ai.get_ai_info {controllerPath:/Game/AI/Guards/BP_GuardAIController}` → `{"controllerClass":"BP_GuardAIController_C"}` (thin)
- `ai.get_ai_info {controllerPath: ...full obj path retry}` → same thin payload

The agent burned an extra `get_ai_info` call probing for whether the empty
perception payload was a path-resolution / transient artifact (it was neither),
because the response gives no signal distinguishing "controller has no perception"
from "this verb doesn't report perception." With nothing in the method summary or
the `ai` overlay flagging that `get_ai_info` only echoes the controller class,
the empty result reads as ambiguous and invites exactly this retry.

## What it should do / how to fix (docs-first)

Lowest-cost fix is to document the limitation on the `docs/wiki-src/ai.md`
overlay so agents don't treat `get_ai_info` as a perception-state read-back:

- Add (or extend) a `## Reading back an AI Controller's perception` section
  stating plainly that `ai.get_ai_info` reports only the controller class and
  does **not** enumerate the AI Perception component or its `SensesConfig`
  (sight/hearing/damage) or team affiliation. Point agents to the supported
  read-back: spawn the controller (`actor.spawn_from_blueprint`) and read
  `actor.get_component_property {componentName, propertyName:"SensesConfig"}`
  (the same control path used to confirm `set_ai_perception` in
  `B-configure-sense-config-silent-noop`), or `blueprint.inspect { assetPath,
  includeProperties:true }` to read the component template's CDO. The
  `### ai.get_ai_info` H3 section should carry the same note inline.

Optional ergonomic follow-up (separate, larger): enrich `ai.get_ai_info` to
walk the controller's SCS for an `AIPerceptionComponent` and emit its
`SensesConfig` (per-sense class + key fields like `SightRadius`,
`LoseSightRadius`, `PeripheralVisionAngleDegrees`, `HearingRange`) and team id,
so a single `ai` verb closes the configure→verify loop without spawning an
instance. Documenting the current behavior is the cheap win; the enrichment is
the nice-to-have.

**Workaround:** confirm perception wiring with
`actor.spawn_from_blueprint` + `actor.get_component_property { componentName:
"AIPerception"|"AIPerceptionComponent", propertyName: "SensesConfig" }`, or
`blueprint.inspect { assetPath, includeProperties: true }` — not
`ai.get_ai_info`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `ai.configure_sight_config` guard-AI perception task (16 calls; all writes `ok:true`). PROCESS friction distinct from the judge-filed `B-configure-sense-config-silent-noop` (silent no-op write): the namespace's own `ai.get_ai_info` read-back returns only `{"controllerClass":"BP_GuardAIController_C"}` and never reports the AI Perception component, `SensesConfig`, sense fields, or team — so the story's "read AI info back to confirm" step is structurally unverifiable through the `ai` surface, and would remain so even after the write no-op is fixed. The thin payload gives no signal distinguishing "no perception" from "verb doesn't report perception," so the agent burned an extra trial-and-error `get_ai_info` retry with the full object path (friction note: "retried get_ai_info with the full object path to rule out a transient empty payload, same thin result"). Same shape as `E-game-framework-info-not-asset-readback`. Docs overlay to improve: `docs/wiki-src/ai.md` (add `## Reading back an AI Controller's perception`; carry the note on `### ai.get_ai_info`). Workaround: confirm via `actor.spawn_from_blueprint` + `actor.get_component_property {propertyName:SensesConfig}` or `blueprint.inspect includeProperties`.
- `#2-additional-blackboard-bt-omission` `OPEN` reporter — Additional evidence (independent realism-mode replay, guard-AI asset-wiring task, separate run): the controller branch of `ai.get_ai_info` omits MORE than just perception — it also never reports the assigned `DefaultBlackboard` / `DefaultBehaviorTree`. REPLAY-CONFIRMED at this HEAD: created BB_OracleGuard + BT_OracleGuard + BP_OracleGuardController, ran `ai.assign_blackboard` and `ai.assign_behavior_tree` (both returned `propertyAssigned:true`), then `ai.get_ai_info {controllerPath:/Game/AIOracle/Controllers/BP_OracleGuardController}` returned exactly `{"aiInfo":{"controllerClass":"BP_OracleGuardController_C"}}` — no blackboard, no behavior tree. Proof the readback is hiding genuinely-set state, not reporting absence: `property.get` on the CDO `Default__BP_OracleGuardController_C` returns `DefaultBlackboard=/Game/AIOracle/Blackboards/BB_OracleGuard.BB_OracleGuard` and `DefaultBehaviorTree=/Game/AIOracle/BehaviorTrees/BT_OracleGuard.BT_OracleGuard`. So the configure→verify thinness this ticket describes for perception applies identically to the BB/BT assignment the same namespace's `assign_*` verbs just made. Root cause is the same code branch: `AIHandler.cpp` `ai.get_ai_info` controller block (~lines 2113-2122) only emits `controllerClass` from `Controller->GeneratedClass->GetName()` and reads no CDO property — whereas the BT branch (`behaviorTreeName`+`hasRootNode`) and the BB branch (`keyCount`+`keys[]`) of the same handler are rich. The optional enrichment in the body (walk the controller's SCS/CDO) should ALSO surface `DefaultBlackboard`/`DefaultBehaviorTree`, not only `SensesConfig`/team; the docs note on `### ai.get_ai_info` should say the controller readback reports only the class and confirm BB/BT/perception wiring via `property.get` on the CDO `Default__<Controller>_C` (`DefaultBlackboard`/`DefaultBehaviorTree`) or `blueprint.scs.get` for the perception component.
- `#4-additional-after-writes-fixed` `OPEN` reporter — Additional evidence (seed-mode replay, seed `ai.configure_hearing_config`, GuardPatrolController guard-AI task, 10 calls all `ok:true`). Confirms the readback thinness is now the SOLE remaining friction once the write no-op is fixed: at this HEAD the sibling `B-configure-sense-config-silent-noop` fix is live, so the `configure_*_config` writes genuinely persist — I REPLAY-CONFIRMED via the SubobjectDataSubsystem that the GuardPatrolController perception template's `SensesConfig` holds `AISenseConfig_Hearing hearing_range=1500.0`, `AISenseConfig_Sight sight_radius=2000.0 lose_sight_radius=2400.0 peripheral_vision_angle_degrees=70.0`, and `AISenseConfig_Damage` (all values exactly as configured; NOT a no-op). Yet `ai.get_ai_info {controllerPath:/Game/AI/Controllers/GuardPatrolController}` STILL returns exactly `{"aiInfo":{"controllerClass":"GuardPatrolController_C"}}` — no perception component, no SensesConfig, no team. So even with the writes fixed (as this ticket predicted in its body: "Even once those writes are fixed, `ai.get_ai_info` still would not surface the perception/sense state"), the story's prescribed "read the controller's AI info back and confirm the hearing range" step is structurally unverifiable through the `ai` surface — the attempt agent reported the SUCCESS CHECK could not pass and had to fall back to a C++ read of the `get_ai_info` handler to understand why. Sibling readback routes confirmed present at this HEAD (`blueprint.scs.get`, `actor.get_component_property`, `blueprint.inspect`, `property.get`) — so this remains ERGONOMIC/discovery, not a hard gap. Reinforces the `#1`/`#2`/`#3` docs steer: the `### ai.get_ai_info` note and a `## Reading back an AI Controller's perception` section on `docs/wiki-src/ai.md` should state the controller readback reports only the class and point to the SCS/CDO read routes for SensesConfig and BB/BT. Sharpens the workaround the `#2` entry proposes: the two CDO read-back routes are NOT interchangeable per field, and the agent paid a dead call discovering it. The scalar controller CDO props worked via the generic reflection route — `property.get` on `Default__BP_GuardController_C` returned `DefaultBlackboard` and `DefaultBehaviorTree` (two successful calls) — but when the agent reached for the SAME shape to read the **perception component** (`property.get { objectPath: ".../BP_GuardController.Default__BP_GuardController_C:AIPerception", propertyName: SensesConfig }`) it got `[OBJECT_NOT_FOUND] Unable to find object at path /Game/AI/Controllers/BP_GuardController.Default__BP_GuardController_C:AIPerception` — the CDO-subobject `:Component` segment does not resolve through the `property.*` route — and only the **next** call, `blueprint.scs.get { AIPerception, SensesConfig }`, succeeded (read back SightRadius=1500/LoseSightRadius=1800/PeripheralVisionAngleDegrees=60). So the configure→verify round-trip cost a wasted misuse-then-correct call (friction note verbatim: *"wasted one call guessing the CDO subobject path for the perception component (OBJECT_NOT_FOUND) before blueprint.scs.get worked. This get_ai_info reporting gap for controllers is a real discoverability issue."*). This reinforces the exact docs steer `#2` proposes and adds the missing precision: the `### ai.get_ai_info` / `## Reading back an AI Controller's perception` note must state that BB/BT are readable via `property.get` on the CDO `Default__<Controller>_C` (scalar props) but the **AI Perception component is NOT** — `property.get` on the `Default__<Controller>_C:AIPerception` subobject path returns `OBJECT_NOT_FOUND`; use `blueprint.scs.get` (or `actor.get_component_property` on a spawned instance) for the component's `SensesConfig` instead. Had the readback verb (or the overlay) carried that note, both the extra `get_ai_info` retry from `#1` and this OBJECT_NOT_FOUND probe would not have happened.
- `#5-docs-fix-in-review` `IN-REVIEW` developer — Implemented the docs-first fix (cheap win) on the `ai.md` overlay; left the optional `get_ai_info` enrichment as the carved-out follow-up. Verified at this HEAD the controller branch is unchanged and still emits only `controllerClass` (`AIHandler.cpp:2235-2240`), the sibling write fix is live, and the overlay had no readback steer — so this is a genuine, unaddressed docs gap (GO, not ALREADY-FIXED). Changes in `Docs/wiki-src/ai.md`: added a `## Reading back an AI Controller's perception` section (renders on the `ai` namespace page) plus a `### ai.get_ai_info` H3 (surfaces when an agent calls `call("ai.get_ai_info")`). Both state plainly that the controller readback reports only the class and never the AI Perception component / `SensesConfig` / team / `DefaultBlackboard` / `DefaultBehaviorTree`, that a thin payload means "verb doesn't report it" (not "no perception") so a retry is pointless, and carry the `#4` precision: BB/BT are readable via `property.get` on the CDO `Default__<Controller>_C` (scalar props) but the **perception component is NOT** — `property.get` on the `:AIPerception` subobject path returns `OBJECT_NOT_FOUND`; use `blueprint.scs.get` (or `actor.get_component_property` on a spawned instance) for `SensesConfig`, or `blueprint.inspect includeProperties` for the template CDO. Regression test added in `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` (`FWikiHandlerGetAiInfoDocumentsThinReadbackTest`, `...wiki_handler.MethodPage.GetAiInfoDocumentsThinReadback`): drives production `WikiHandler::RenderPage("ai.get_ai_info")` and asserts the rendered page contains the overlay-exclusive markers `SensesConfig`, `blueprint.scs.get`, and `OBJECT_NOT_FOUND` (none appear in the auto summary or param list) — it fails iff the overlay H3/section is reverted. Did not compile/run (later phase).
- `#6-additional-evidence-guard-bb-bt` `IN-REVIEW` reporter — Additional corroborating evidence (independent struggle audit, BB_Guard/BT_Guard/BP_GuardController guard-AI wiring task, ~29 calls, all `ok:true` apart from two expected probe failures). Reconfirms at this HEAD the controller-branch thinness this ticket (esp. `#2`/`#4`) describes for BB/BT AND perception: `ai.get_ai_info {controllerPath:/Game/AI/BP_GuardController}` returned ONLY `{controllerClass}` (friction note verbatim: *"ai.get_ai_info's controllerPath branch returns ONLY {controllerClass} - it cannot confirm the controller's blackboard/behavior-tree/perception, so the success check's premise that get_ai_info reports them is impossible"*). The agent fell back exactly as the `#2`/`#4` workaround prescribes — `property.get` on the CDO `Default__BP_GuardController_C` for `DefaultBlackboard` (→ BB_Guard) and `DefaultBehaviorTree` (→ BT_Guard, after wiring), and `blueprint.scs.get {nameMatch:Perception}` for the perception `SensesConfig` (SightRadius=1500/LoseSightRadius=1600). The agent still had to read the plugin C++ (`AIHandler.cpp`) "as a last resort" to understand the thin readback — so even at this HEAD the discoverability cost is being paid; the `#5` `ai.md` overlay (once live) would short-circuit that. Independent reinforcement that the controller `get_ai_info` readback gap is the dominant verification friction on guard-AI wiring tasks and that BOTH workaround routes (`property.get` CDO for BB/BT, `blueprint.scs.get` for perception) are the ones agents actually reach for.
- `#7-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
