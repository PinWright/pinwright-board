---
id: F-anim-bind-asset-on-player-node
title: "No typed RPC to bind UAnimSequence/UBlendSpace/UPoseAsset to player nodes"
status: DONE
severity: High
category: feature
tags: [animation, anim-graph, asset-binding, sequence-player, blend-space-player, imperative-api]
---

# No typed asset-binding RPC for `AnimGraphNode_*Player` nodes

The single most common operation on an AnimGraph player node — pointing
`AnimGraphNode_SequencePlayer` at a `UAnimSequence`,
`AnimGraphNode_BlendSpacePlayer` at a `UBlendSpace`,
`AnimGraphNode_PoseByName` at a `UPoseAsset`,
`AnimGraphNode_AimOffsetPlayer` at a `UAimOffsetBlendSpace` — has no
typed peer in the imperative API.

The only escape hatch is `animation.authoring.set_anim_graph_node_value`
with `propertyName: "Sequence"` (or `"BlendSpace"`, or `"PoseAsset"`)
and a string `value`. That handler is string-typed: it doesn't validate
that the asset class matches the property's expected typed pointer, it
doesn't resolve a soft path to the loaded object, and it gives no
signal to the caller about *which* property name on *which* runtime
node holds the binding. Empirically the property paths differ:

| AnimGraph node | Runtime struct | Asset property |
|----------------|----------------|----------------|
| `AnimGraphNode_SequencePlayer` | `FAnimNode_SequencePlayer` | `Sequence` (`UAnimSequence*`) |
| `AnimGraphNode_BlendSpacePlayer` | `FAnimNode_BlendSpacePlayer` | `BlendSpace` (`UBlendSpace*`) |
| `AnimGraphNode_BlendSpaceEvaluator` | `FAnimNode_BlendSpaceEvaluator` | `BlendSpace` |
| `AnimGraphNode_PoseByName` / `_PoseBlendNode` | `FAnimNode_PoseHandler` | `PoseAsset` (`UPoseAsset*`) |
| `AnimGraphNode_AimOffsetLookAt` | `FAnimNode_AimOffsetLookAt` | `BlendSpace` (`UAimOffsetBlendSpace*`) |
| `AnimGraphNode_LinkedAnimGraph` | `FAnimNode_LinkedAnimGraph` | `InstanceClass` (`UClass<UAnimInstance>*`) + `Tag` |
| `AnimGraphNode_LinkedAnimLayer` | `FAnimNode_LinkedAnimLayer` | `Layer` (`FName`) + `InstanceClass` |
| `AnimGraphNode_ControlRig` | `FAnimNode_ControlRig` | `ControlRigClass` (`UClass<UControlRig>*`) |

Agents must already know each row to use `set_anim_graph_node_value`,
and the handler doesn't surface "wrong asset type" errors at the
asset-class layer — the BP compiler catches it later as a generic
"can't autocast" message.

Additionally, the same player nodes carry **asset-player options** —
`PlayRate`, `bLoopAnimation`, `StartPosition`, `bTeleportToExplicitTime`,
play-rate basis — most of which need "Expose as Pin" to be set first
(see [`F-anim-expose-pin-on-anim-node`](F-anim-expose-pin-on-anim-node.md))
if the caller wants a variable binding rather than a static default.
There is no single RPC that says "create a sequence player for asset X
with loop=true and play rate=1.2", forcing 3+ round-trips per player
(create_node → set_anim_graph_node_value for asset → set_anim_graph_node_value
for loop → set_anim_graph_node_value for rate).

**Use cases blocked:**

1. Authoring a state's pose graph imperatively without trial-and-error
   on property names.
2. Verifying that the asset class is correct *before* compile fails.
3. One-shot player authoring (most common AnimBP operation).

**Workaround:** Drop to AGIR (`call("anim.compile_agir")`) which has
typed sequence-player / blend-space-player statements. The cliff
completion notes confirm both round-trip cleanly.

**Proposal:** Add `animation.authoring.bind_player_asset` (params:
`blueprintPath`, `graphName`, `nodeId`, `assetPath`, `options?: { loop?:
bool, playRate?: number, startPosition?: number }`, `save?`). Resolves
the node, dispatches on the wrapped `FAnimNode_*` struct to pick the
correct asset-property name and expected `UClass`, validates the loaded
asset is that class (returning `ASSET_CLASS_MISMATCH` with the expected
class on failure), sets the property, optionally writes the asset-player
options. Mirrors AGIR's typed sequence-player coverage.

Also add a one-shot factory:
`animation.authoring.add_sequence_player(blueprintPath, graphName,
assetPath, x, y, loop?, playRate?)` → equivalent to `create_node` +
`bind_player_asset` in one transaction. Same shape for
`add_blend_space_player`, `add_pose_by_name`, `add_linked_anim_graph`.

Implementation surface: each `UAnimGraphNode_*` derived class exposes
its runtime struct as a public `FAnimNode_* Node` member;
`FAnimNode_AssetPlayerBase` (and direct subclasses) declare the asset
pointer as a `UPROPERTY(EditAnywhere)` with a known name. The
class → property-name table can be hardcoded for the ~12 player-style
classes that matter; a fallback for unrecognized classes falls through
to the existing `set_anim_graph_node_value` behavior.

**Cross-ref:**
[`F-search-api-anim-graph-nodes`](F-search-api-anim-graph-nodes.md)
proposes the discovery layer (`requiresAsset` field in the search
result). This ticket is the corresponding mutation layer.
[`F-anim-expose-pin-on-anim-node`](F-anim-expose-pin-on-anim-node.md)
covers the variable-bind half of the same workflow.

## History
- `#1-no-typed-asset-bind` `OPEN` reporter — Only path to set the bound asset on a `AnimGraphNode_SequencePlayer`/`BlendSpacePlayer`/`PoseByName`/`AimOffset`/`LinkedAnimGraph`/`ControlRig` node is `set_anim_graph_node_value` with a string property name and value. No asset-class validation, no docs on which property name maps to which runtime node, no one-shot factory combining `create_node` + asset bind + asset-player options. Proposes `animation.authoring.bind_player_asset` plus per-class factories (`add_sequence_player`, `add_blend_space_player`, etc.) that mirror AGIR's typed coverage.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Verified against `Handlers/Animation/AnimationAuthoringHandler.cpp` (only `set_anim_graph_node_value` at line 2813; no `bind_player_asset`, `add_sequence_player`, `add_blend_space_player`, etc.) and `Handlers/Blueprint/BlueprintGraphHandler.cpp` (`blueprint.graph.set_node_property` is also string-typed with no asset-class validation). Severity High is defensible — imperative AnimBP authoring is a common workflow and the current path forces a hidden property-name lookup table plus 3+ round-trips per player. Note: a validation-only overlay on `set_anim_graph_node_value` would not solve the one-shot factory case or the property-name discoverability problem; the new typed RPC(s) are the right shape. Additional finding: the existing handler resolves `propertyName` via `FoundNode->GetClass()->FindPropertyByName(...)` on the outer `UAnimGraphNode_*`, not on the inner `FAnimNode_*` struct stored under `Node` — so the current escape hatch may not even reach the asset UPROPERTY without a path expression, strengthening the case for a dedicated handler.
- `#3-shipped-bind-player-asset` `IN-REVIEW` developer — Added `animation.authoring.bind_player_asset` in AnimationAuthoringHandler.cpp dispatching on the engine's `UAnimGraphNode_AssetPlayerBase::SetAnimationAsset` virtual (avoiding the proposed per-class table — the virtual is overridden on all 8 player/evaluator classes). Validates expected asset class via `GetAnimationAssetClass()`, applies optional `{loop, playRate, startPosition}` through `WriteAnimNodeFieldByName`. Per-class factories (`add_sequence_player` etc.) and UClass-binding for LinkedAnimGraph/ControlRig deferred — they bind a UClass not a UAnimationAsset and need a separate ticket. Regression test FAnimAuthoringBindPlayerAssetTest covers the SequencePlayer path.
- `#4-verify-bind-player-asset` `DONE` tester — Verified: created temp AnimSequence and AnimBlueprint, added `AnimGraphNode_SequencePlayer`, then ran `animation.authoring.bind_player_asset` with options `{loop:true, playRate:1.2, startPosition:0.1}`; observed `success: true`, `expectedClass: "AnimSequenceBase"`, `boundAssetClass: "AnimSequence"`, and `optionsApplied: 3`.
