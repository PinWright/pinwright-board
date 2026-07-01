---
id: F-anim-sync-group-on-player-nodes
title: "Sync groups on AssetPlayer / BlendSpace nodes have no typed authoring"
status: DONE
severity: Medium
category: feature
tags: [animation, anim-graph, sync-group, asset-player, imperative-api]
---

# No typed RPC to set sync group / role on AnimGraph player nodes

`animation.authoring.add_sync_marker` covers the per-sequence side of
sync (named markers on a `UAnimSequence`), but the AnimGraph-side
binding — telling a `SequencePlayer` / `BlendSpacePlayer` /
`BlendSpaceEvaluator` / `PoseByName` *which* sync group it participates
in and in which role — lives on the player node itself, not on the
asset.

The runtime data is on `FAnimNode_AssetPlayerBase::GroupName` (`FName`)
and `FAnimNode_AssetPlayerBase::GroupRole`
(`EAnimGroupRole::Type` — `CanBeLeader`, `AlwaysLeader`,
`AlwaysFollower`, `TransitionLeader`, `TransitionFollower`). On older
UE branches the same fields hide behind a wrapping `FAnimationGroupReference Sync`
struct. There is no typed setter today: callers have to drop to
`animation.authoring.set_anim_graph_node_value` on every player node
with the right (version-dependent) property path, no enum-name parsing
for `GroupRole`, and no class validation that the node is actually a
`UAnimGraphNode_AssetPlayerBase` descendant. Sync-group authoring is
the missing half of "make these two players blend in lockstep".

**Use cases blocked:**

1. Authoring a locomotion blend space where the `Locomotion` sync group
   binds the lower-body player as leader and the upper-body player as
   follower, fully imperatively.
2. State-machine entries that pin specific players as
   `TransitionLeader` for cross-state phase matching.
3. Setting up an additive overlay player to follow the base layer's
   sync group without dropping to AGIR.

**Workaround:** The imperative path is two
`set_anim_graph_node_value` calls per node with version-conditional
property names and a magic enum integer.

**Proposal:** Add
`animation.authoring.set_sync_group(blueprintPath, graphName?, nodeName,
groupName, role?, save?)` where:

- `nodeName` must resolve to a `UAnimGraphNode_AssetPlayerBase`
  descendant. Anything else → `NODE_NOT_ASSET_PLAYER` with the actual
  class name in the error payload.
- `groupName` is an `FName`; empty string clears the binding (and the
  handler must reject empty name with `role != Standalone` →
  `INVALID_SYNC_GROUP_PAIR`).
- `role` is a string mapped to `EAnimGroupRole::Type`. Accept the enum
  literal names (`"CanBeLeader"`, `"AlwaysLeader"`, `"AlwaysFollower"`,
  `"TransitionLeader"`, `"TransitionFollower"`) plus a synthetic
  `"Standalone"` that means "no sync group" (clears `GroupName`).
  Default: `"CanBeLeader"` to match engine default.

Implementation surface: walks `Node->GetFNodeProperty()` to reach the
wrapped `FAnimNode_*`, casts through the common
`FAnimNode_AssetPlayerBase` base, and writes through
`SetGroupName`/`SetGroupRole` so subclasses own the version-specific
details.
Pairs with [`F-anim-add-graph-node-generic`](F-anim-add-graph-node-generic.md):
once that one-shot factory exists, `syncGroup`/`syncRole` should be
accepted as keys in its `properties?` dict and routed through the same
typed setter before generic property reflection, so newly-spawned
players can take sync-group binding in their initial RPC.

**Cross-ref:**
[`F-anim-bind-asset-on-player-node`](F-anim-bind-asset-on-player-node.md)
covers the asset half of the same player-node authoring story; this
ticket is the sync-side peer.
[`F-rpc-animation-list-sync-markers`](F-rpc-animation-list-sync-markers.md)
covers the per-sequence introspection side.

## History
- `#1-no-typed-sync-group-setter` `OPEN` reporter — `animation.authoring.add_sync_marker` covers per-sequence markers, but the AnimGraph-side sync binding (`FAnimNode_AssetPlayerBase::GroupName` + `GroupRole`) has no typed authoring path. Today: raw `set_anim_graph_node_value` per node, version-conditional property name, no enum mapping, no class validation. Proposes `animation.authoring.set_sync_group(blueprintPath, graphName, nodeId, groupName, role)` with enum-name role mapping, `NOT_ASSET_PLAYER_NODE` rejection for non-player nodes, and `INVALID_SYNC_GROUP_PAIR` rejection for empty name with non-Standalone role. Pairs with `F-anim-add-graph-node-generic` so its `properties` dict can take `syncGroup` / `syncRole` keys.
- `#2-reformulate-virtual-setters` `OPEN` developer — Reframed implementation around UE 5.6 `FAnimNode_AssetPlayerBase` `SetGroupName`/`SetGroupRole` virtuals, corrected `nodeId` to existing `nodeName` convention, corrected error code to `NODE_NOT_ASSET_PLAYER`, removed the unsupported current-source claim that AGIR already has `syncGroup`/`syncRole` aliases, and included `add_graph_node` properties support because that factory is already present.
- `#3-set-sync-group-handler` `IN-REVIEW` developer — Added `animation.authoring.set_sync_group` and `add_graph_node` `syncGroup`/`syncRole` support in `AnimationAuthoringHandler.cpp`, documented the typed sync-group flow, and added `FAnimAuthoringSetSyncGroupTest` coverage.
- `#4-verify-sync-group` `DONE` tester — Verified: created `/Game/EditorAutomationTests/ABP_McpVerify_F_anim_sync_group`, `animation.authoring.add_graph_node` applied `syncGroup`/`syncRole` with no property failures, and `animation.authoring.set_sync_group` returned `groupName=Overlay`, `role=AlwaysFollower`, `syncMethod=SyncGroup`; temp asset deleted afterward.
