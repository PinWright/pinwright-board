---
id: F-bt-composite-child-order-not-authorable
title: "No `behavior_tree.*` verb can set or change a composite child's execution order — order is graph-X position, `add_node` is the only verb that writes it, and it cannot be re-written"
status: IN-REVIEW
severity: High
category: feature
tags: [behavior-tree, authoring, node-position, execution-order, selector, sequence, node-guid, decompile]
encounters: 2
lastSeen: 2026-09-07T08:05:00Z
---

# A Behavior Tree's execution order cannot be edited after the node exists

In UE a composite's children run **left to right by graph X position**, not by connection order.
`behavior_tree.decompile` reports that position (`@(x, y)` on every node), so the order is
*readable*. It is not *writable* on an existing node:

- `behavior_tree.add_node` takes `x` and `y` — and required, with the docstring
  "required — nodes stack at origin if all callers pass 0", so the surface already knows position
  is load-bearing. It only applies at creation.
- `behavior_tree.set_node_properties` takes `nodeId`, `comment` and `properties` (node *instance*
  properties). No `x`/`y`.
- `behavior_tree.connect_nodes` takes `parentNodeId` and `childNodeId`. No index, no
  before/after sibling.
- `behavior_tree.break_connections` + `connect_nodes` re-parents but does not move, so a
  reconnected child keeps its old X and therefore its old priority.

So the only route to "run this leaf first" is `remove_node` on the subtree and `add_node` it back
at a smaller X — which throws away the node's instance properties, its decorators and its services,
all of which then have to be re-authored from the decompile by hand.

## Where it bit (this checkout, 2026-09-07)

`/Game/FPS/AI/BT_Enemy`, the ammo branch:

    child composite Sequence @(-700, 320) {
      decorator Blackboard (BlackboardKey: AmmoState, GreaterOrEqual, IntValue: 1)
      child task RunEQSQuery  @(-900, 520)   (BlackboardKey: CoverLocation)
      child task MoveTo       @(-700, 520)   (BlackboardKey: CoverLocation, AcceptableRadius: 60)
      child task BTT_TakeCover@(-500, 520)
      child task BTT_Reload   @(-300, 520)
    }

`BTT_Reload` is last, so the AI must *reach* cover before it reloads. A 90 s three-enemy probe
recorded 0 reload starts in 528 samples: all three emptied a 30-round magazine, none reloaded, and
the reserve stayed at 180 — because the `MoveTo` ahead of the reload never completed. The one-line
fix is "reload first, then move": give `BTT_Reload` an X left of `-900`. There is no verb that can
do it.

Reordering a Selector's *branches* has the same problem and is the more common case — branch
priority is exactly what a BT author tunes, and every tuning pass needs to move an existing subtree
left or right.

## Ask

Either:

1. `behavior_tree.set_node_properties` accepts `x` / `y` alongside `comment` (smallest change; the
   handler already writes those fields on the `add_node` path), **or**
2. `behavior_tree.connect_nodes` accepts `childIndex` (or `beforeChildId` / `afterChildId`) and the
   handler assigns the X that produces that order, **or**
3. a dedicated `behavior_tree.set_child_order` taking `parentNodeId` and an ordered `childNodeIds`
   array — the form that makes the intent explicit and cannot leave two children at the same X.

Whichever lands, the response should echo the resulting order the way `decompile` prints it, so a
caller can verify the priority it just asked for rather than re-decompiling to check.

## The documented workaround does not work either: node GUIDs are undiscoverable

Attempted on this tree, 2026-09-07:

    behavior_tree.remove_node {assetPath:"/Game/FPS/AI/BT_Enemy", nodeId:"BTT_Reload_C_0"}
    -> [NODE_NOT_FOUND] Node not found.

`remove_node` takes a "Node GUID". `behavior_tree.decompile` is the only read verb for a tree and
it emits **no GUIDs** — its nodes are printed by class, label and `@(x, y)` position, and the object
paths it does print (`BT_Enemy:BTT_Reload_C_0`) are runtime node object names, which `remove_node`
rejects. `decompile` takes `assetPath` and nothing else, so there is no flag to ask for ids.

The consequence is wider than this ticket's own workaround. **Every** `behavior_tree` verb keyed on
`nodeId` — `remove_node`, `set_node_properties`, `attach_decorator`, `attach_service`,
`break_connections`, `connect_nodes` — is reachable only for nodes whose id the caller still holds
from an `add_node` call in the same session. On a tree authored earlier, in another session, or by
hand in the editor, none of them can be addressed at all. A BT is a long-lived asset that gets tuned
repeatedly; that is the normal case, not the edge case.

`decompile` printing each node's GUID alongside its position would unblock this ticket, the
workaround, and every other nodeId-keyed verb at once, and is a smaller change than any of the three
asks above.

## Workaround

None. `remove_node` + `add_node` at a new X was the intended fallback, and it cannot be issued
because the node cannot be named. Editing the tree by hand in the Unreal editor is the only route,
which defeats the point of the authoring surface.

## Notes

- Distinct from `E-ai-bt-authoring-verbs-dead-end`, which is about the deprecated `ai.*` verbs
  producing orphaned, unaddressable nodes. This one is on the supported `behavior_tree.*` surface
  and is about editing a tree that is already correct in structure but wrong in priority.
- Distinct from `B-bt-set-node-properties-silent-noop`: passing `x` to `set_node_properties` today
  is a declared-param failure, not a silent drop.

## History

- **#2, 2026-09-07, AI stream.** Attempted the documented remove-and-re-add workaround and found it
  unreachable: `behavior_tree.decompile` emits no node GUIDs and `remove_node` rejects the runtime
  node object name (`NODE_NOT_FOUND`). **Severity stays High.** The reach is not a
  second stream — the AI stream is the only one authoring Behavior Trees in this project, and that
  half of the test is not met. It is the verb surface: node GUIDs being undiscoverable blocks
  `remove_node`, `set_node_properties`, `attach_decorator`, `attach_service`, `break_connections`
  and `connect_nodes` on **any** tree not created in the current session, which is every tree that
  has ever been saved. The original ask (make order editable) is now the smaller half of the
  ticket; emitting GUIDs from `decompile` unblocks all six verbs at once.
- **#2 severity corrected, 2026-09-07.** The `High -> Critical` bump in the entry above was wrong
  and is reverted: per the board README, Critical is reserved for an editor crash or asset data
  loss. Reach and cost move a ticket **within** its impact class, and a hard blocker with no
  workaround tops out at High. This one destroys no data and crashes nothing — it makes an
  authoring operation impossible — so High is the ceiling and the reach argument does not lift it.
- `#3-emit-node-ids-and-set-child-order` `IN-REVIEW` developer — Both halves fixed.
  (a) **Ids.** `BTIRDecompiler::BuildGraphNodeFields` (`Source/PinWright/Private/BTIR/BTIRDecompiler.cpp`)
  now leads every graph-node-backed field list with `nodeId: <guid>` — `FGuid::ToString()`, byte-identical
  to what `add_node` returns and what `FindBTGraphNode` matches — so composites, tasks, attached
  decorators/services, the composite-decorator wrapper and the `root_aux` block are all addressable
  from a decompile alone. Inner conditions of a composite decorator get none (they live in a bound
  sub-graph and no `nodeId` verb reaches them). `btir.txt` gained an aspect-version row (1 → 2) in
  `Private/Handlers/Asset/AssetDumpCache.cpp` so cached dumps regenerate. `FindBTGraphNode`
  (`Private/Handlers/AI/BehaviorTreeHandler.cpp`) gained a **second** pass matching the NodeInstance
  object name (`BTT_Reload_C_0`, the name this ticket's repro was rejected on); it runs only after
  every graph-node identity has missed, so a GUID always wins and existing resolution is unchanged.
  (b) **Order.** New verb `behavior_tree.set_child_order {assetPath, parentNodeId, childNodeIds[]}`
  in the same file: `childNodeIds` must be a permutation of the parent's current children (a short,
  long, duplicated or foreign list is rejected with the parent's `currentChildOrder`), the children's
  own existing X values are re-dealt in the requested order under one `FScopedTransaction` (ties
  pushed apart by 200), and the rebuild goes through the engine's own
  `UBehaviorTreeGraph::RebuildChildOrder`, which sorts each output pin's `LinkedTo` with
  `FCompareNodeXLocation` and calls `UpdateAsset(KeepRebuildCounter)` — the same routine the BT editor
  runs on a node drag, not a reimplementation. The response echoes `childOrder` read back off the
  runtime `UBTCompositeNode::Children` (falling back to pin order for the Root entry node and for an
  orphaned parent), each entry carrying `nodeId`, the `name` decompile prints, and `x`/`y` matching
  its `@(x, y)`. Docs: new `## Node ids` and `## Child execution order` sections plus the `nodeId:`
  grammar in the decompile section of `docs/wiki-src/behavior_tree.md`, and the `btir.txt` line in
  `docs/wiki-src/asset.dump-sidecars.md`. Tests (new file
  `Source/PinWright/Private/Tests/Gameplay/TestBehaviorTreeChildOrder.cpp`):
  `PinWright.behavior_tree.decompile.EmitsResolvableNodeIds` builds a tree through the RPCs, scrapes
  every `nodeId:` out of the decompile **text**, asserts the scraped set contains each id `add_node`
  returned, and feeds each scraped string back to `set_node_properties` — revert the decompiler change
  and the scrape returns nothing, so the count and containment assertions fail;
  `PinWright.behavior_tree.set_child_order.ReordersCompositeChildren` asserts the runtime composite
  runs left-then-right, calls `set_child_order` with the order reversed, and asserts both the echoed
  `childOrder` and `UBTCompositeNode::Children` now start with the formerly-second task — revert the
  verb and the invoke finds no handler. Not compiled or run here (orchestrator's phase).
  **Verifier follow-up, same entry:** `set_child_order` now refuses any parent with more than one
  output pin, with `INVALID_ARGUMENT` and an `outputPinCount` field. `UBehaviorTreeGraphNode_SimpleParallel`
  is the one BT node with two (`Task` + `Out`, `BehaviorTreeGraphNode_SimpleParallel.cpp` ~28-29) and the
  engine sorts each pin's `LinkedTo` independently, so the flattened cross-pin list `CollectBTLinkedChildren`
  produced validated as a permutation, moved the X positions, and left both child arrays unchanged while
  reporting success — a silent no-op with an echoed order that had not changed. Documented in the wiki-src
  `## Child execution order` section; regression test
  `PinWright.behavior_tree.set_child_order.RefusesMultiOutputPinParent` (same file) builds a real
  `UBehaviorTreeGraphNode_SimpleParallel` by reflection (the class carries no export macro, and `add_node`
  cannot make one — it builds every composite on the single-pin `_Composite` graph node), guards that the
  fixture really has two output pins, wires one Wait task to each pin directly (`connect_nodes` only ever
  uses the first output pin), and asserts the swapped-order call comes back `bSuccess == false` /
  `INVALID_ARGUMENT` / `outputPinCount == 2` — revert the guard and the call succeeds instead.
  Deliberately left alone: the Root entry node still gets no BTIR line when it carries no
  decorators/services, so on a tree from an earlier session its id is still not readable out of
  `decompile` — that gap is `E-bt-root-entry-node-undiscoverable`'s, which fixed it by returning
  `rootNodeId` from `create`, and widening the `root_aux` emission here would collide with it.
