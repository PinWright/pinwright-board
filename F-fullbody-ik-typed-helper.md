---
id: F-fullbody-ik-typed-helper
title: "No typed authoring for FullBody IK / TwoBoneIK / LegIK skeletal control nodes"
status: DONE
severity: Medium
category: feature
tags: [animation, anim-graph, skeletal-control, ik, imperative-api, ergonomic]
---

# Skeletal-control IK nodes have no typed authoring path

`animation.authoring.add_ik_chain` (AnimationAuthoringHandler.cpp:4464)
operates on a `UIKRigDefinition` asset — it adds a named chain entry
to the IK Rig data asset. It does NOT author the in-graph skeletal
control nodes that consume IK at runtime inside an AnimBlueprint:

- `UAnimGraphNode_RigUnit_FullbodyIK` (Animation/IKRig plugin)
- `UAnimGraphNode_TwoBoneIK`
- `UAnimGraphNode_LegIK`
- `UAnimGraphNode_ModifyBone`
- `UAnimGraphNode_RotateRootBone`

These nodes fall into the long tail of ~120 `UAnimGraphNode_*` classes
covered by the generic
[`F-anim-add-graph-node-generic`](F-anim-add-graph-node-generic.md)
handler (`animation.authoring.add_graph_node`). That generic path
spawns the node and writes property dicts, but skeletal-control nodes
carry **nested struct properties** that are awkward over the generic
text-keyed property route:

- `FBoneReference` (bone name + cached index + per-bone-space enum)
  appears as `IKBone`, `EffectorBone`, `JointTargetBone`,
  `BoneToModify`, etc.
- `FBoneSocketTarget` (target bone OR socket, with auxiliary fields)
  for effector targeting.
- `TArray<FRigUnit_FullbodyIK_Effector>` (FullbodyIK) — array of
  nested structs each containing a `FBoneReference`, transform, pull
  weights, and rotation/position blend modes.
- Per-axis space enums (`EBoneControlSpace`) for translation /
  rotation / scale on `ModifyBone`.

Callers authoring a foot-IK pass today have to either:
1. Use `add_graph_node` with deeply-nested property JSON for each
   `FBoneReference` field and hope the generic property-write path
   recurses through nested structs (currently inconsistent on
   `TArray<FStruct>`).
2. Author via AGIR — works, but mixes ergonomics levels for callers
   that are otherwise on the imperative path.

**Use cases blocked / painful:**

1. One-shot authoring of a `TwoBoneIK` foot-IK node — typical pattern
   is `IKBone=foot_l`, `EffectorBone=foot_l`,
   `JointTargetBone=knee_l_jointtarget`, `Alpha=1.0` exposed as pin.
2. `ModifyBone` for procedural head-look or weapon-aim corrections
   (translation/rotation in component or parent-bone space, alpha
   from a curve).
3. `FullbodyIK` effector arrays for cinematic poses (most painful —
   nested-struct-array authoring is the weakest part of the generic
   property route).
4. `LegIK` quick setup for quadruped/biped foot-planting passes.

**Workaround:** Use `animation.authoring.add_graph_node` with the
generic property dictionary, or compile via AGIR. The generic path
works for flat properties but is verbose; nested-struct-array writes
(FullbodyIK effectors) currently need AGIR.

**Proposal:** Add typed helpers for the two most common cases first
and defer FullbodyIK to a follow-up once its effector-array shape
stabilizes:

- `animation.authoring.add_two_bone_ik` (params: `blueprintPath`,
  `graphName`, `ikBone`, `effectorBone`, `jointTargetBone`,
  `effectorLocationSpace?`, `jointTargetLocationSpace?`, `x`, `y`,
  `alpha?`, `save?`).
- `animation.authoring.add_modify_bone` (params: `blueprintPath`,
  `graphName`, `boneName`, `translation?`, `rotation?`, `scale?`,
  `translationSpace?`, `rotationSpace?`, `scaleSpace?` (each one of
  `BCS_WorldSpace`/`BCS_ComponentSpace`/`BCS_ParentBoneSpace`/
  `BCS_BoneSpace`), `translationMode?`/`rotationMode?`/`scaleMode?`
  (`Ignore`/`Replace`/`Additive`), `x`, `y`, `alpha?`, `save?`).

`add_leg_ik` and `add_fullbody_ik` follow the same pattern once these
two land — keep the API surface conservative and add helpers as real
demand surfaces (per CLAUDE.md: YAGNI; no speculative generalization).

Resolution shape: each helper resolves the node class, calls
`FGraphNodeCreator<UAnimGraphNode_*>`, writes the
`FBoneReference.BoneName` directly (cached index repopulates on
compile), sets the space enums, then `ReconstructNode()` +
`MarkBlueprintAsStructurallyModified` per the existing pattern in
AnimationAuthoringHandler.cpp. Validates that bone names exist on the
target skeleton; returns `BONE_NOT_FOUND` on miss (the AnimBP's
`TargetSkeleton` is reachable via the graph's owning blueprint).

**Cross-ref:** Pairs with
[`F-anim-add-graph-node-generic`](F-anim-add-graph-node-generic.md)
(generic fallback) and
[`F-anim-expose-pin-on-anim-node`](F-anim-expose-pin-on-anim-node.md)
(for exposing `Alpha` / per-effector pins on these nodes). Same
nested-bone-ref problem appears on `CopyBone`, `ObserveBone`,
`SplineIK` — solve those the same way once the two flagship helpers
prove the pattern.

## History
- `#1-no-typed-ik-helpers` `OPEN` reporter — `animation.authoring.add_ik_chain` operates on `UIKRigDefinition` asset, NOT on in-graph skeletal control nodes. AnimGraph IK / bone-modify nodes (`AnimGraphNode_RigUnit_FullbodyIK`, `TwoBoneIK`, `LegIK`, `ModifyBone`, `RotateRootBone`) fall into the ~120-class long tail covered only by the recently-shipped generic `animation.authoring.add_graph_node` ([F-anim-add-graph-node-generic](F-anim-add-graph-node-generic.md)). Their nested-struct properties (`FBoneReference` per effector, `FBoneSocketTarget`, `TArray<FRigUnit_FullbodyIK_Effector>`, per-axis `EBoneControlSpace` enums, translation/rotation modes) are awkward over the generic property-dict path — especially the FullbodyIK effector array, where nested-struct-array writes route through AGIR today. Proposes typed `add_two_bone_ik(blueprintPath, graphName, ikBone, effectorBone, jointTargetBone, x, y)` and `add_modify_bone(blueprintPath, graphName, boneName, translation/rotation/scale spaces, alpha, x, y)` as the two most common cases. Defer FullbodyIK and LegIK helpers until real demand surfaces (YAGNI). Implementation reuses the `FGraphNodeCreator` + `ReconstructNode` + `MarkBlueprintAsStructurallyModified` pattern from the existing 8 typed helpers in AnimationAuthoringHandler.cpp; validates bone names against the AnimBP's TargetSkeleton.
- `#2-add-ik-node-helpers` `IN-REVIEW` developer — Added typed `animation.authoring.add_two_bone_ik` and `animation.authoring.add_modify_bone` handlers in `AnimationAuthoringHandler.cpp`, documented the skeletal-control helper flow, and added `FAnimAuthoringAddSkeletalControlIkHelpersTest` coverage.
- `#3-verify-typed-helpers` `DONE` tester — Verified: created temp AnimBlueprint `/Game/App/UI/Test/W_McpVerifyTemp_F_fullbody_ik_typed_helper`, ran `animation.authoring.add_two_bone_ik` with `foot_l` / `calf_l` and observed `success: true`, `nodeClass: AnimGraphNode_TwoBoneIK`, typed bone and space fields echoed; ran `animation.authoring.add_modify_bone` with `spine_01` and observed `success: true`, `nodeClass: AnimGraphNode_ModifyBone`, typed mode/space/alpha fields echoed; deleted the temp asset with `asset.delete`.
