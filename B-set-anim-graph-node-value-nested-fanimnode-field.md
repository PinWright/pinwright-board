---
id: B-set-anim-graph-node-value-nested-fanimnode-field
title: "set_anim_graph_node_value can't reach nested FAnimNode_* struct fields (Alpha, bone refs, spaces) — flat outer-node lookup only"
status: IN-REVIEW
severity: Medium
category: bug
tags: [animation, anim-graph, set-anim-graph-node-value, two-bone-ik, skeletal-control, nested-property]
---

# `set_anim_graph_node_value` only sees direct UPROPERTYs on the outer `UAnimGraphNode_*`, never the `FAnimNode_*` struct under `Node`

`animation.authoring.set_anim_graph_node_value` is documented as "Set a
property value on an AnimGraph node by name" and its wiki Notes say "Use
this for static defaults". But the handler resolves the property with a
single flat lookup on the **outer** editor node class:

```cpp
// AnimationAuthoringHandler_AnimBlueprint.cpp:1890
FProperty* Property = FoundNode->GetClass()->FindPropertyByName(FName(*PropertyName));
if (!Property) { Ctx.SendError(TEXT("PROPERTY_NOT_FOUND"), ...); return true; }
```

Almost every configurable default on a stock AnimGraph node does **not**
live as a direct UPROPERTY on the `UAnimGraphNode_*` class — it lives on
the wrapped runtime struct exposed as the public `FAnimNode_* Node`
member (e.g. `FAnimNode_TwoBoneIK::Alpha`, the bone references, the
`*LocationSpace` fields, `FAnimNode_ModifyBone::Translation`,
`FAnimNode_AssetPlayerBase::Sequence/PlayRate/GroupName`). Because the
handler never descends into the `Node` struct member, those fields are
all unreachable — `set_anim_graph_node_value` returns `PROPERTY_NOT_FOUND`
for valid, documented property names. There is also no struct-path
traversal: `propertyName: "Node.Alpha"` is fed verbatim to
`FindPropertyByName` and also fails.

This is a known shortcoming that was already noted (but not fixed for this
handler) on the board: see
[`F-anim-bind-asset-on-player-node`](F-anim-bind-asset-on-player-node.md)
history `#2-reviewed-and-confirmed` — "the existing handler resolves
`propertyName` via `FoundNode->GetClass()->FindPropertyByName(...)` on the
outer `UAnimGraphNode_*`, not on the inner `FAnimNode_*` struct stored
under `Node` — so the current escape hatch may not even reach the asset
UPROPERTY without a path expression". Those tickets shipped **new typed
RPCs** (`bind_player_asset`, `set_sync_group`) and gave the generic
factory `add_graph_node` a runtime-FAnimNode-field fallback (see
[`F-anim-add-graph-node-generic`](F-anim-add-graph-node-generic.md)
history `#5/#6`), all routing through `AnimGraphConstructionUtils`'
`ResolveAnimNodeFieldByName` / `PinWright::Anim::GetFNodeProperty` /
`GetFNode`. But the bread-and-butter **mutate-an-existing-node** verb,
`set_anim_graph_node_value`, never gained that fallback. The result is an
asymmetry: the create-time helpers can write nested fields, the generic
post-hoc setter cannot.

Concrete impact (the task that surfaced this): after creating a
`TwoBoneIK` node with `animation.authoring.add_two_bone_ik` (which sets
`alpha` at creation via `Node->Node.Alpha`), there is **no** way to later
re-set that alpha. `add_two_bone_ik`/`add_modify_bone` only set alpha at
creation; there is no `set_two_bone_ik` mutator; and the generic
`set_anim_graph_node_value` can't reach `Alpha`. So "create the node, then
soften its alpha to 0.8" is impossible via the imperative surface — the
only remaining route is a full AGIR rewrite.

**Verbatim repro** (against `/Game/ExampleContent/IKRig/Anim/ABP_DinoDragon_FootIK`,
an anim BP with a `Two Bone IK - Bone: ankle_R` node created by
`add_two_bone_ik`):

```
call("animation.authoring.set_anim_graph_node_value", {
  blueprintPath: "/Game/ExampleContent/IKRig/Anim/ABP_DinoDragon_FootIK",
  nodeName: "ankle_R", propertyName: "Alpha", value: "0.8" })
-> [PROPERTY_NOT_FOUND] Property 'Alpha' not found on node 'ankle_R'

call(... propertyName: "Node.Alpha" ...)
-> [PROPERTY_NOT_FOUND] Property 'Node.Alpha' not found on node 'ankle_R'
```

`Alpha` is genuinely a field of `FAnimNode_TwoBoneIK` (the same field
`add_two_bone_ik` writes as `Node->Node.Alpha` at
AnimationAuthoringHandler_AnimBlueprint.cpp:2463), so the input is valid;
it is the handler's flat lookup that rejects it.

**Workaround:** Drop to AGIR (`call("anim.compile_agir")`) to rewrite the
node, or set the value at creation time only. For player-node asset/sync
fields, use the dedicated `bind_player_asset` / `set_sync_group` typed
RPCs.

**Fix:** When the flat `GetClass()->FindPropertyByName` misses, fall back
to the runtime-FAnimNode-field path the siblings already use
(`AnimGraphConstructionUtils::ResolveAnimNodeFieldByName` over
`GetFNodeProperty`/`GetFNode`), and/or accept a `Node.<field>` /
`<field>` struct path. This is the same fallback `add_graph_node` got in
`F-anim-add-graph-node-generic` #5; applying it to
`set_anim_graph_node_value` makes the generic post-hoc setter able to edit
the nested defaults (Alpha, bone refs, spaces, translations) that are the
whole point of skeletal-control / player nodes.

## History
- `#1-initial-repro` `OPEN` reporter — `animation.authoring.set_anim_graph_node_value` resolves `propertyName` only via `FoundNode->GetClass()->FindPropertyByName` on the outer `UAnimGraphNode_*` (AnimationAuthoringHandler_AnimBlueprint.cpp:1890), so it cannot reach fields on the wrapped `FAnimNode_* Node` struct where almost all configurable defaults live. Replay-confirmed against `/Game/ExampleContent/IKRig/Anim/ABP_DinoDragon_FootIK`: setting `Alpha` (and `Node.Alpha`) on a `TwoBoneIK` node both return `[PROPERTY_NOT_FOUND] ... not found on node 'ankle_R'`, even though `Alpha` is a real `FAnimNode_TwoBoneIK` field that `add_two_bone_ik` writes via `Node->Node.Alpha` at creation. There is no `set_two_bone_ik` mutator, so re-setting a TwoBoneIK node's alpha after creation is impossible via the imperative surface. Sibling helpers `add_graph_node`/`bind_player_asset`/`set_sync_group` already route nested-field writes through `AnimGraphConstructionUtils::ResolveAnimNodeFieldByName` (`GetFNodeProperty`/`GetFNode`) — this generic setter never got that fallback (noted but not fixed in `F-anim-bind-asset-on-player-node` #2). Fix: add the same runtime-FAnimNode-field fallback (and/or a `Node.<field>` struct path) to `set_anim_graph_node_value`.
- `#2-fix` `IN-REVIEW` developer — Confirmed the defect is live in current source: the `set_anim_graph_node_value` handler still does only the flat `FoundNode->GetClass()->FindPropertyByName` lookup with a hard `PROPERTY_NOT_FOUND` bail (AnimationAuthoringHandler_AnimBlueprint.cpp:1890-1894), no descent into the `FAnimNode_*` `Node` struct. Implemented the same fallback the sibling `add_graph_node` already uses (same file, ~2816-2828): when the flat lookup misses, `Cast<UAnimGraphNode_Base>(FoundNode)` and call `AnimGraphConstructionUtils::ApplyJsonValueToAnimNodeFieldByName(AnimNode, FName(*PropertyName), ValueField, ...)`, which routes through `ResolveAnimNodeFieldByName` → `GetFNodeProperty`/`GetFNode` and applies via the same `ApplyJsonValueToProperty` path; on success it marks the BP modified, saves, and sends the same success result. Only on miss of BOTH paths does it return `PROPERTY_NOT_FOUND`. Files: `Source/PinWright/Private/Handlers/Animation/AnimationAuthoringHandler_AnimBlueprint.cpp`. Regression test added in `Source/PinWright/Private/Tests/Assets/TestAnimGraphHandlers.cpp` (`FAnimAuthoringSetGraphNodeValueNestedFieldTest`, "PinWright.anim.authoring.SetGraphNodeValueNestedField"): authors a TwoBoneIK node via `add_two_bone_ik` (alpha 0.75), then dispatches `set_anim_graph_node_value` propertyName `Alpha` value `0.8` through the real RpcDispatcher and asserts the dispatch succeeds (not `PROPERTY_NOT_FOUND`) and the runtime `Node.Alpha` is now 0.8 — fails if the fallback is reverted. Did not compile/run (a later phase verifies).
