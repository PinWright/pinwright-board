---
id: F-bt-attach-decorator-service-to-parent
title: "Decorators and services have no RPC to attach to a parent composite node"
status: DONE
severity: High
category: feature
tags: [behavior-tree, ai, decorator, service, parent-binding, imperative-api]
---

# No way to bind a decorator or service to a parent composite node

Behavior Tree decorators and services are not standalone graph nodes
in runtime — they live as `Decorators[]` / `Services[]` arrays on a
parent `UBehaviorTreeGraphNode` (composite or task). The current API
has no RPC that models this attachment:

- `ai.add_decorator` (`AIHandler.cpp` line 951) creates a
  `UBTDecorator_*` via `NewObject<UBTDecorator_…>(BT)` and immediately
  drops the reference. The resulting object is orphaned — not in any
  composite's `Decorators` array, not in the graph, not serialized
  anywhere reachable. The RPC reports success but produces no visible
  effect on the asset.
- `ai.add_service` (line 1006) is worse: it doesn't even allocate the
  service object. It just dirties the package and returns a "reference
  created" message.
- `behavior_tree.add_node` with `nodeType: "Decorator"` / `"Service"`
  creates a `UBehaviorTreeGraphNode_Decorator` or `_Service` as a
  standalone graph node, but doesn't add it to any parent composite's
  `Decorators` / `Services` array — the editor renders these floating
  graph nodes but the runtime BT ignores them.

There is no `parentNodeId` parameter on any of these calls, so even
when the agent knows which composite the decorator should hang off of,
there is no way to express it.

**Use cases blocked:**
1. Imperatively building any non-trivial behavior tree — decorators
   gate composite execution, services tick on composite entry, and
   neither is useful detached.
2. Round-tripping an existing BT (decompile → edit → recompile) — the
   decompiler can read the parent-bound structure but the imperative
   API can't emit it.
3. Adding a `UBTDecorator_Blackboard` to a sequence to gate it on a
   blackboard key (the canonical BT idiom).

**Workaround:** Edit the asset's `UBehaviorTree::RootNode` ->
composite `Children[].DecoratorOps` / `.Decorators` /
`.Services` arrays directly via
`asset.set_property` or a property-edit RPC. Brittle: requires
agents to know the exact UPROPERTY array path and the
`UBTCompositeChild` nested struct layout. Easy to leave the graph
editor view out of sync with the runtime arrays.

**Fix:** Add two new BT-namespaced handlers:
- `behavior_tree.attach_decorator` — params `assetPath`,
  `parentNodeId` (composite or task GUID),
  `decoratorClass` (path or short name), `x?`, `y?`, optional
  `properties`. Resolves the parent graph node, creates the
  `UBehaviorTreeGraphNode_Decorator` wrapping the decorator class,
  appends to `Parent->Decorators` and to the graph, calls
  `NotifyGraphChanged`. Returns the new decorator node GUID.
- `behavior_tree.attach_service` — same shape but for
  `UBTService_*` and `UBehaviorTreeGraphNode_Service`, attaching to
  `Parent->Services`. Tasks can also hold services in some UE
  versions; handler should accept both composite and task parents
  and error with `INVALID_PARENT` otherwise.

Both should accept Blueprint-derived classes (e.g.
`UBTService_BlueprintBase` subclasses created by the sibling ticket
[`F-bt-create-blueprint-node-classes`](F-bt-create-blueprint-node-classes.md))
via `ResolveClassByName`, matching the resolution path
`behavior_tree.add_node` already uses (BehaviorTreeHandler.cpp:247).

Deprecate `ai.add_decorator` / `ai.add_service` or rewrite them
as parent-required aliases that delegate to the new handlers — the
current behavior is silently broken.

**Cross-ref:**
[`F-bt-create-blueprint-node-classes`](F-bt-create-blueprint-node-classes.md)
covers authoring custom BTTask/BTService/BTDecorator BP subclasses.
Together the two tickets close the custom-BT authoring loop.

## History
- `#1-decorator-service-orphaned` `OPEN` reporter — `ai.add_decorator` allocates a decorator but doesn't append it to any parent composite's `Decorators` array (AIHandler.cpp:951); `ai.add_service` doesn't even allocate; `behavior_tree.add_node` with `Decorator`/`Service` `nodeType` creates a floating graph node with no parent binding. No `parentNodeId` parameter exists on any of these. Proposes `behavior_tree.attach_decorator` / `attach_service` with explicit `parentNodeId`, deprecating the broken `ai.*` handlers.
- `#2-attach-bt-subnodes` `IN-REVIEW` developer — Added `behavior_tree.attach_decorator` and `behavior_tree.attach_service` with explicit parent binding through BehaviorTree graph subnode semantics, disabled the broken `ai.add_decorator` / `ai.add_service` success paths, rejected floating decorator/service creation from `behavior_tree.add_node`, fixed regular BT node class data before placement, and added `FBTAttachDecoratorServiceToParentTest`.
- `#3-fix-iteration-2` `IN-REVIEW` developer — Fixed attach review issues: short native decorator/service names now resolve through `/Script/AIModule.BTDecorator_*` or `BTService_*` before broad class-name lookup, the handler no longer wraps UE's transactional `UAIGraphNode::AddSubNode`, and failed subnode instance creation removes the inserted subnode and rebuilds graph-derived BT state before returning an error.
- `#4-verify-attached-subnodes` `DONE` tester — Verified: created `/Game/McpTests/BehaviorTree/BT_McpVerifyTemp_FBtAttachDecoratorService`, added `Wait` node `298D8B4841C9B9F684474380CBEB55CC`, ran `behavior_tree.attach_decorator` with `decoratorClass:"Blackboard"` and `behavior_tree.attach_service` with `serviceClass:"DefaultFocus"` against that parent, and `behavior_tree.decompile` emitted both `decorator Blackboard` and `service DefaultFocus` nested inside the `orphan task Wait` block; temp asset deleted.
