---
id: F-bt-composite-child-order-not-authorable
title: "No `behavior_tree.*` verb can set or change a composite child's execution order — order is graph-X position, `add_node` is the only verb that writes it, and it cannot be re-written"
status: OPEN
severity: Critical
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
  node object name (`NODE_NOT_FOUND`). **Severity High -> Critical by reach.** The reach is not a
  second stream — the AI stream is the only one authoring Behavior Trees in this project, and that
  half of the test is not met. It is the verb surface: node GUIDs being undiscoverable blocks
  `remove_node`, `set_node_properties`, `attach_decorator`, `attach_service`, `break_connections`
  and `connect_nodes` on **any** tree not created in the current session, which is every tree that
  has ever been saved. The original ask (make order editable) is now the smaller half of the
  ticket; emitting GUIDs from `decompile` unblocks all six verbs at once.
