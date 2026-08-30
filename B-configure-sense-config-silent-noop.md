---
id: B-configure-sense-config-silent-noop
title: "`ai.configure_sight_config` / `configure_hearing_config` / `configure_damage_sense_config` are silent no-op stubs — echo the input but never write SensesConfig"
status: IN-REVIEW
severity: High
category: bug
tags: [ai, perception, ai-perception-component, sight, hearing, damage, silent-noop, sense-config, success-no-effect]
---

# The `ai.configure_*_config` perception verbs echo success but never touch the perception component

`ai.configure_sight_config` is documented as "Configure sight sense on a blueprint's perception component". In practice it is a no-op stub: it loads the Blueprint, copies the input `sightConfig` numbers straight into the response JSON, marks the package dirty, and returns `{"message":"Sight sense configured", ...echoed values}` — **without ever locating the `AIPerceptionComponent`, creating a `UAISenseConfig_Sight`, or adding anything to `SensesConfig`**. Same for `ai.configure_hearing_config` (echoes `hearingRange`) and `ai.configure_damage_sense_config` (doesn't even read a config — just marks dirty and returns "Damage sense configured"). The success-shaped echo makes the no-op indistinguishable from a real write, and there is no readback verb on the `ai` surface that exposes sense config (`ai.get_ai_info` returns only `{"controllerClass":"..._C"}`), so a caller doing the natural "configure → read back to confirm" round-trip can neither apply nor observe the change.

This is the root cause of the friction in the seed-mode fuzz task that built `BP_GuardAIController`: every write call echoed its values, but the perception component had zero sense configs and `ai.get_ai_info` could confirm nothing.

## Root cause (AIHandler.cpp)

- `ai.configure_sight_config` (lines ~998–1036): after `LoadObject<UBlueprint>`, the body only does `Result->SetNumberField(sightRadius/loseSightRadius/peripheralVisionAngle, ...)` from the input, `Blueprint->MarkPackageDirty()`, and `SendSuccess`. No `AIPerceptionComponent` lookup, no `NewObject<UAISenseConfig_Sight>`, no `SensesConfig` mutation.
- `ai.configure_hearing_config` (lines ~1038–1070): same shape — echoes `hearingRange`, marks dirty, returns success. No write.
- `ai.configure_damage_sense_config` (lines ~1072–1094): reads no config at all; just `MarkPackageDirty()` + `"Damage sense configured"`.

Contrast the **working** implementation in the same file — `ai.set_ai_perception` / `ai.setup_perception` (lines ~2171–2248 and ~2670–2723) actually do `NewObject<UAISenseConfig_Sight>(PerceptionComp)` and add to the array — which is why that path persists and the `configure_*` path does not.

## Replay-confirmed repro (live editor, `mcp__editor-automation__call`)

Clean isolated repro on a fresh asset:

1. `ai.create_ai_controller {name:"BP_SightOnly", path:"/Game/AI/SightProbe"}` → created.
2. `ai.add_ai_perception_component {blueprintPath:"/Game/AI/SightProbe/BP_SightOnly"}` → `{"componentName":"AIPerception", ...}` (component persists; confirmed via `asset.dump` scs.json).
3. `ai.configure_sight_config {blueprintPath:"/Game/AI/SightProbe/BP_SightOnly", sightConfig:{sightRadius:1500, loseSightRadius:1800, peripheralVisionAngle:70}}`
   → `{"sightRadius":1500,"loseSightRadius":1800,"peripheralVisionAngle":70,"message":"Sight sense configured", ...existsAfter:true}` (success-shaped echo).
4. `actor.spawn_from_blueprint {blueprintPath:"/Game/AI/SightProbe/BP_SightOnly", actorName:"SightOnlyInst"}` → spawned.
5. `actor.get_component_property {actorName:"SightOnlyInst", componentName:"AIPerception", propertyName:"SensesConfig"}`
   → `{"value": []}`  **(empty — the sight sense was never persisted)**

Control proving the read surface works and isolating the bug to the `configure_*` family: the same `actor.get_component_property ... SensesConfig` read on a controller configured via `ai.set_ai_perception {enableSight:true, sightRadius:1500, ...}` returns `{"value":[{}]}` (one populated entry). So `[]` after `configure_sight_config` is a genuine no-op, not a reflection artifact.

## Impact

The discovery-schema-obvious path to wire perception onto an AI Controller — `add_ai_perception_component` then `configure_sight_config` / `configure_hearing_config` / `configure_damage_sense_config` — produces a perception component with no senses while reporting success for every step. A caller has no JSON-RPC signal that nothing landed. (`ai.set_perception_team` at lines ~1096+ is the same stub shape — echoes `teamId`, marks dirty, no write — and should be reviewed alongside these three.)

**Workaround:** use `ai.set_ai_perception` / `ai.setup_perception` (`{controllerPath, enableSight, sightRadius, loseSightRadius, peripheralVisionAngle, enableHearing, hearingRange, enableDamage}`), which actually creates the sense configs and adds them to `SensesConfig`. Note that path creates a perception component named `AIPerceptionComponent` (vs `add_ai_perception_component`'s `AIPerception`).

**Fix options (any of):**
1. **Implement them.** Resolve the Blueprint's `AIPerceptionComponent` template from the SCS (`FindSCSNode` / `GetActualComponentTemplate`, mirroring how `set_ai_perception` reaches the component), `NewObject<UAISenseConfig_Sight/Hearing/Damage>(PerceptionComp)`, populate the fields (`SightRadius`, `LoseSightRadius`, `PeripheralVisionAngleDegrees`, hearing `HearingRange`), add to `SensesConfig` (replacing any existing config of the same class), then mark dirty + recompile so the template change persists onto spawned instances. Reuse the `set_ai_perception` sense-construction code.
2. **Make them fail loud.** Until implemented, replace `SendSuccess` with `SendError("NOT_IMPLEMENTED", "configure_sight_config does not write the perception component yet — use ai.set_ai_perception")`. Removes the silent-success footgun and steers callers to the working verb. (Mirrors the resolution of `B-material-stub-handlers-silent-success`.)

## History
- `#1-initial-repro` `OPEN` reporter — Seed-mode fuzz of `ai.configure_sight_config` (guard-AI perception wiring task). Replay-confirmed on fresh `/Game/AI/SightProbe/BP_SightOnly`: `add_ai_perception_component` + `configure_sight_config {1500/1800/70}` returned success echoing the values, but `actor.spawn_from_blueprint` + `actor.get_component_property {componentName:AIPerception, propertyName:SensesConfig}` returned `[]`. Control via `ai.set_ai_perception` on a sibling asset returned `SensesConfig:[{}]`, proving the read surface works and isolating the no-op to the `configure_*_config` family. Root cause read from `AIHandler.cpp` lines ~998–1094: all three handlers only echo input + `MarkPackageDirty()`, never touch the `AIPerceptionComponent` / `SensesConfig` (vs the working `set_ai_perception` at ~2171/2670 which `NewObject`s the sense configs). `ai.set_perception_team` (~1096) shares the stub shape. Not a duplicate of the DONE silent-success tickets (`B-material-stub-handlers-silent-success`, other namespaces) nor of `B-stop-behavior-tree-noop-wrong-var` / `E-ai-assign-verbs-need-set-default-fallback` (BT/blackboard assign/stop verbs, not perception sense config).
- `#2-implemented-real-writes` `IN-REVIEW` developer — Fixed via Option 1 (real implementation reusing the `set_ai_perception` sense-construction pattern) in `Source/EditorAutomationRpcGateway/Private/Handlers/AI/AIHandler.cpp`. Added a file-local `FindPerceptionComponentTemplate(UBlueprint*)` helper that resolves the `UAIPerceptionComponent` template from the blueprint's SCS. Rewrote all three handlers: `ai.configure_sight_config` now `NewObject<UAISenseConfig_Sight>(PerceptionComp)` + sets SightRadius/LoseSightRadius/PeripheralVisionAngleDegrees/affiliation/MaxAge + `ConfigureSense()`; `ai.configure_hearing_config` does the same with `UAISenseConfig_Hearing` (HearingRange); `ai.configure_damage_sense_config` with `UAISenseConfig_Damage`. All three now resolve the component (returning new error `PERCEPTION_COMPONENT_NOT_FOUND` instead of fake-success when none exists), then `MarkBlueprintAsStructurallyModified` + `McpSafeAssetSave` so the template change persists. `ai.set_perception_team` (flagged same-shape) left as-is — out of scope per ticket. Regression test added in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAIHandlers.cpp`: `EditorAutomationRpcGateway.ai.configure_sight_config.WritesSenseConfig` creates a controller, adds the perception component, calls `configure_sight_config {1500/1800/70}`, reloads the blueprint, and asserts `PerceptionComp->GetSenseConfig<UAISenseConfig_Sight>()` is non-null with the persisted radii/angle — this fails against the old no-op stub (which left SensesConfig empty). Not compiled/tested here; a later phase verifies.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
