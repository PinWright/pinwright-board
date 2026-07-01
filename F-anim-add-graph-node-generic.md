---
id: F-anim-add-graph-node-generic
title: "Named anim-graph-node helpers cover ~8 of ~120 classes; no generic typed creator"
status: DONE
severity: Medium
category: feature
tags: [animation, anim-graph, imperative-api, ergonomic, node-creation]
---

# `animation.authoring.*` only knows about a handful of AnimGraph node classes

`animation.authoring.*` exposes typed creators for a small set of
AnimGraph node classes:

- `add_state_machine` → `UAnimGraphNode_StateMachine`
- `add_slot_node` → `UAnimGraphNode_Slot`
- `add_blend_node` (`blendType: "TwoWayBlend" | "LayeredBlend"`) →
  `UAnimGraphNode_TwoWayBlend` / `UAnimGraphNode_LayeredBoneBlend`
- `add_layered_blend_per_bone` → `UAnimGraphNode_LayeredBoneBlend`
  (duplicate path with one bone param; see
  [`F-anim-layered-blend-bone-weights`](F-anim-layered-blend-bone-weights.md))
- `add_cached_pose` → `UAnimGraphNode_SaveCachedPose`
- Plus state-machine internals (`add_state`, `add_transition`)

That's ~8 of the ~120 stock `UAnimGraphNode_*` classes (see
[`F-search-api-anim-graph-nodes`](F-search-api-anim-graph-nodes.md)).
Every other node type — `SequencePlayer`, `BlendSpacePlayer`,
`BlendSpaceEvaluator`, `PoseByName`, `PoseBlendNode`, `ApplyAdditive`,
`MakeDynamicAdditive`, `LinkedAnimGraph`, `LinkedAnimLayer`,
`ControlRig`, `RigidBody`, `ModifyBone`, `TwoBoneIK`, `ObserveBone`,
`SkeletalControl`, `RotateRootBone`, `BlendListByEnum`,
`BlendListByBool`, `BlendListByInt`, `BlendByPosePlayer`,
`BlendMultiplyByFloat`, `CopyBone`, `LegIK`, `IKRig`, etc. — has no
typed authoring path.

The fallback is `blueprint.graph.create_node` with `nodeType:
"AnimGraphNode_SequencePlayer"`. Empirically that works — the
generic K2-spawn factory accepts `UAnimGraphNode_*` classes — but the
created node has **no pins exposed** (no `PlayRate`, `Sequence`,
`Alpha`, etc.) and no asset bound. The caller then needs:

1. `set_anim_graph_pin_exposed` (not yet implemented — see
   [`F-anim-expose-pin-on-anim-node`](F-anim-expose-pin-on-anim-node.md))
   for every property they want as a pin.
2. `bind_player_asset` (not yet implemented — see
   [`F-anim-bind-asset-on-player-node`](F-anim-bind-asset-on-player-node.md))
   for the asset binding.
3. `set_anim_graph_node_value` for each static default.

So even with the four tickets in this audit landed, callers who want a
node not in the ~8-class helper set still go through several round
trips per node. AGIR statements (`call("anim.compile_agir")`) collapse
all of that into one text statement per node.

**Use cases blocked:**

1. Authoring `AnimGraphNode_ApplyAdditive` / `MakeDynamicAdditive` /
   `BlendListByEnum` imperatively without dropping to AGIR.
2. Setting up skeletal control nodes (`ModifyBone`, `TwoBoneIK`,
   `RotateRootBone`, `LegIK`) where the bone reference + space + alpha
   are all node-defaults.
3. Linked-anim authoring with `Layer` + `InstanceClass` together (AGIR
   has typed `linked_anim` coverage; imperative path is N round-trips).

**Workaround:** Use `blueprint.graph.create_node` plus the helpers
proposed in the sibling tickets; or compile an AGIR document covering
the whole subgraph.

**Proposal:** Add `animation.authoring.add_graph_node` (params:
`blueprintPath`, `graphName`, `nodeClass`, `x`, `y`, `properties?: {
[name]: jsonValue }`, `exposePins?: string[]`, `bindAsset?: string`,
`save?`). One transaction that:

1. Spawns the named `UAnimGraphNode_*` (validates it's a descendant of
   `UAnimGraphNode_Base`, returns `INVALID_ANIM_NODE_CLASS` otherwise).
2. Writes the property dictionary, recursing into nested struct
   properties.
3. Toggles `bShowPin=true` for every entry in `exposePins` and reconstructs.
4. Resolves `bindAsset` against the node's expected asset class.

Reduces the typical "author one sequence player with PlayRate exposed
and Sequence bound" workflow from 4 round trips to 1. For complex
nested struct properties (e.g. `LayerSetup`), callers fall through to
the typed helpers like `set_layered_blend_layers`.

Implementation surface: walks
`TObjectIterator<UClass>` once at registration (or reuses the catalog
from [`F-search-api-anim-graph-nodes`](F-search-api-anim-graph-nodes.md))
to build a class-name → `UClass*` map for `UAnimGraphNode_Base`
descendants, then dispatches per call. Property writes use the same
`PropertyUtils` path the existing `set_anim_graph_node_value` uses.

**Cross-ref:** Pairs with the four other anim-imperative tickets. Once
all five land, an agent can author any greenfield AnimGraph without
touching AGIR, at the cost of more verbose call sequences than the
text-IR. AGIR remains the elegant path for bulk authoring; this ticket
makes the imperative path complete.

## History
- `#1-named-helpers-cover-eight-classes` `OPEN` reporter — `animation.authoring.*` has typed helpers for ~8 of ~120 `UAnimGraphNode_*` classes; `blueprint.graph.create_node` accepts the rest but produces nodes with no exposed pins / no asset bound. Authoring one `SequencePlayer` with `PlayRate` exposed and `Sequence` bound is 4 round trips today (create_node + expose_pin + bind_asset + set property). Proposes one-shot `animation.authoring.add_graph_node` accepting `nodeClass` + `properties` + `exposePins` + `bindAsset`. Implementation reuses the catalog from `F-search-api-anim-graph-nodes` and pairs with `F-anim-expose-pin-on-anim-node` / `F-anim-bind-asset-on-player-node` / `F-anim-state-machine-internals` / `F-anim-layered-blend-bone-weights` to close the imperative gap vs AGIR.
- `#2-reviewed-and-confirmed` `OPEN` tester — Verified by source inspection. AnimationAuthoringHandler.cpp registers 7 distinct anim-graph-node creators (`add_state_machine`, `add_state`, `add_transition`, `add_blend_node` covering TwoWayBlend+LayeredBoneBlend, `add_cached_pose`, `add_slot_node`, `add_layered_blend_per_bone`) — matches the "~8" claim. UE 5.6 `Source/Editor/AnimGraph/Public` contains ~80 `AnimGraphNode_*` headers alone; with `Plugins/Animation/*` (Warping, BlendStack, ControlRig, IKRig, LiveLink, PoseSearch, RigLogic) the total comfortably approaches the ticket's ~120 figure. Fallback claim is correct: `blueprint.graph.create_node`'s dynamic path (BlueprintGraphHandler.cpp:1288) does `NewObject` + `PostPlacedNewNode` + `AllocateDefaultPins` only — no `bShowPin` toggling, no asset binding. Severity Medium is appropriate (AGIR exists; imperative-only callers eat 4 round trips, which is ergonomic pain not a blocker). Documentation-only resolution is **insufficient** because the sibling primitives `set_anim_graph_pin_exposed` and `bind_player_asset` aren't yet implemented — until those land, no docs can wire the missing capability. This ticket is correctly scoped as the ergonomic capstone over those siblings.
- `#3-shipped-add-graph-node-generic` `IN-REVIEW` developer — Added `animation.authoring.add_graph_node` in AnimationAuthoringHandler.cpp. Resolves nodeClass via `ClassUtils::ResolveClassByName`, validates `IsChildOf(UAnimGraphNode_Base)`, reuses `AnimGraphConstructionUtils::CreateAnimNode`. Optional `properties` writes go through existing `ApplyJsonValueToProperty`; `exposePins` delegates to the new `ToggleOptionalPinExposed` helper (sibling ticket); `bindAsset` dispatches on `UAnimGraphNode_AssetPlayerBase::SetAnimationAsset` for player nodes with a reflection-fallback for non-player FObjectProperty asset slots. Regression test FAnimAuthoringAddGraphNodeGenericTest creates a SequencePlayer at (100,200) and asserts on node-count + position.
- `#4-returned-property-write-misses-runtime-field` `OPEN` tester — Returned: `animation.authoring.add_graph_node` created `AnimGraphNode_SequencePlayer` at (100,200), exposed `PlayRate`, and bound `/App/ThirdPerson/Characters/Animations/Manny/MM_Idle.MM_Idle`, but `properties: {"PlayRate": 1.25}` came back as `propertiesFailed: [{"name":"PlayRate","error":"property not found"}]` while still reporting `success: true`. Test: created temp `/Game/EditorAutomationTests/ABP_McpVerifyTemp_FAnimAddGraphNodeGeneric_20260515_01` with `animation.authoring.create_anim_blueprint`, called `animation.authoring.add_graph_node` with `nodeClass: "AnimGraphNode_SequencePlayer"`, `exposePins: ["PlayRate"]`, `bindAsset`, and the property payload, then deleted the temp asset with `asset.delete` (`deletedCount: 1`, `existsAfter: false`).
- `#5-runtime-node-properties` `IN-REVIEW` developer — Added a JSON-value runtime FAnimNode field helper in AnimGraphConstructionUtils and updated `animation.authoring.add_graph_node` so property writes fall back from editor-node UObject properties to runtime node fields such as `FAnimNode_SequencePlayer::PlayRate`. Extended `FAnimAuthoringAddGraphNodeGenericTest` to assert `PlayRate` is applied, not failed, exposed as a pin, preserves syncGroup/syncRole, binds the compatible sequence, and leaves the authored SequencePlayer at (100,200).
- `#6-verify-runtime-property-fallback` `DONE` tester — Verified: created temp `/Game/EditorAutomationTests/ABP_McpVerifyTemp_FAnimAddGraphNodeGeneric` (skeleton `/App/ThirdPerson/Characters/Meshes/SK_Mannequin`), then called `animation.authoring.add_graph_node` with `nodeClass:"AnimGraphNode_SequencePlayer"`, `x:100`, `y:200`, `properties:{"PlayRate":1.25}`, `exposePins:["PlayRate"]`, `bindAsset:"/App/ThirdPerson/Characters/Animations/Manny/MM_Idle.MM_Idle"`. Response: `success:true`, `propertiesApplied:["PlayRate"]`, `propertiesFailed:[]`, `pinsExposed:["PlayRate"]`, `assetBound:true`, `nodePosX:100`, `nodePosY:200`, `nodeName:"Sequence Player 'MM_Idle'"`. The #4 regression (PlayRate landing in `propertiesFailed` with "property not found") is gone — the runtime FAnimNode field fallback now resolves `FAnimNode_SequencePlayer::PlayRate`. Cleaned up with `asset.delete` (`deletedCount:1`, `existsAfter:false`).
