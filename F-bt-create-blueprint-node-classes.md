---
id: F-bt-create-blueprint-node-classes
title: "No RPC to author UBTTask/UBTService/UBTDecorator Blueprint subclasses"
status: DONE
severity: High
category: feature
tags: [behavior-tree, ai, blueprint, asset-authoring, btask, btservice, btdecorator]
---

# No RPC to create custom BT task/service/decorator Blueprint subclasses

`behavior_tree.add_node` already accepts arbitrary class paths via the
`ResolveClassByName(NodeType)` branch in `BehaviorTreeHandler.cpp`
(lines 247–271), so once a Blueprint subclass of `UBTTask_BlueprintBase`,
`UBTService_BlueprintBase`, or `UBTDecorator_BlueprintBase` exists, it can
be placed into the graph by passing its class name as `nodeType`. The
gap is one step earlier: there is no RPC to **author** such a Blueprint
in the first place. `blueprint.create` makes a generic
`UBlueprint`/`UAnimBlueprint`/`UWidgetBlueprint`, but Behavior Tree
Blueprints use the dedicated `UBehaviorTreeGraph` editor pipeline and
the `BlueprintFactory` path with the BT-base parent classes — the AI
namespace has no equivalent factory wrapper.

Result: agents cannot script the full "write a custom BTTask, give it
ReceiveExecute logic, drop it into a tree" loop end-to-end. They must
either hand-author the asset in the editor first, or use only the
engine-stock task set (`Wait`, `MoveTo`, `RotateToFaceBBEntry`,
`RunBehavior`, `FinishWithResult`).

**Use cases blocked:**
1. Fully scripted authoring of game-specific AI behaviors.
2. CI/test fixtures that need a known custom BTTask present.
3. Round-tripping a behavior tree from a spec into a fresh project.

**Workaround:** Use `blueprint.create` with `parentClass` set to
`/Script/AIModule.BTTask_BlueprintBase` (and `BTService_BlueprintBase` /
`BTDecorator_BlueprintBase`) — the generic factory accepts these
parents. Add `ReceiveExecuteAI` / `ReceiveTick` event overrides via
`blueprint.add_event` + BPIR. This works but is undocumented for the
BT case; a dedicated handler would surface the right parent classes
and validate that the override events resolve.

**Fix:** Add `behavior_tree.create_task_blueprint`,
`behavior_tree.create_service_blueprint`,
`behavior_tree.create_decorator_blueprint` in `BehaviorTreeHandler.cpp`.
Each takes `name`, `savePath?`, optional `parentClass` (default
`UBTTask_BlueprintBase` etc.), returns the new asset path. Internally
thin wrapper over `KismetEditorUtilities::CreateBlueprint` with the
correct parent — same shape as `blueprint.create` but pinned to the
BT-base parent so agents don't have to know the full
`/Script/AIModule.BTTask_BlueprintBase` path. After creation, the
existing `behavior_tree.add_node` already accepts the new class via
`ResolveClassByName`, so no change to placement is required.

**Cross-ref:**
[`F-bt-attach-decorator-service-to-parent`](F-bt-attach-decorator-service-to-parent.md)
covers the orthogonal gap of binding decorators/services to a parent
composite node — together the two tickets close the custom-BT
authoring loop.

## History
- `#1-no-bt-bp-factory` `OPEN` reporter — `blueprint.create` can produce a BTTask/BTService/BTDecorator BP via the generic parent-class path, but the AI namespace has no typed factory and no documented workflow. `behavior_tree.add_node` already accepts custom-class names through `ResolveClassByName` (BehaviorTreeHandler.cpp:247-271), so once the asset exists placement is solved — the missing piece is authoring. Proposes `behavior_tree.create_task_blueprint` / `create_service_blueprint` / `create_decorator_blueprint` as thin wrappers over `KismetEditorUtilities::CreateBlueprint` pinned to the right BT-base parent.
- `#2-bt-blueprint-factories` `IN-REVIEW` developer — Added typed `behavior_tree.create_task_blueprint`, `behavior_tree.create_service_blueprint`, and `behavior_tree.create_decorator_blueprint` factories with BT BlueprintBase defaults; added `FBehaviorTreeCreateBlueprintNodeClassesTest` to verify registration, success, loadable `UBlueprint` assets, and parent classes.
- `#3-fix-iteration-2` `IN-REVIEW` developer — Added an explicit transaction and `Modify()` calls around `behavior_tree.add_node` graph mutation, including the new graph node and its node instance after placement, without changing the typed Blueprint factory API.
- `#4-verify-bt-factories` `DONE` tester — Verified: live `behavior_tree.create_task_blueprint`, `behavior_tree.create_service_blueprint`, and `behavior_tree.create_decorator_blueprint` calls created saved temp assets under `/Game/McpVerify` with parent classes `/Script/AIModule.BTTask_BlueprintBase`, `/Script/AIModule.BTService_BlueprintBase`, and `/Script/AIModule.BTDecorator_BlueprintBase`; cleanup via `asset.delete` deleted all 3 temp assets.
