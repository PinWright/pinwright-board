---
id: E-add-two-bone-ik-no-effector-location
title: "`add_two_bone_ik` requires a target bone and exposes no effector/joint location, so component-space IK cannot be authored through the verb"
status: DONE
severity: Medium
category: enhancement
tags: [animation, authoring, two-bone-ik]
costly: 1
---

# `add_two_bone_ik` requires a target bone and exposes no effector/joint location, so component-space IK cannot be authored through the verb

`animation.authoring.add_two_bone_ik` takes `ikBone`, `effectorBone`, `jointTargetBone` (all
**required**) plus `effectorLocationSpace` / `jointTargetLocationSpace`. It does **not** take
`effectorLocation` or `jointTargetLocation` — the actual offsets that decide where the limb goes.

Two consequences:

1. **The offsets must be set outside the verb.** After calling it you still need Python
   (`node.get_editor_property('node')` → `set_editor_property('effector_location', ...)` → write the
   struct back), so the typed helper does not actually finish the node it creates. That is the
   opposite of the stated purpose ("use this typed helper for skeletal-control nodes whose nested
   runtime fields are awkward through `add_graph_node` property dictionaries").

2. **There is no way to express "no target bone".** `FAnimNode_TwoBoneIK`'s `EffectorTarget` /
   `JointTarget` accept an *empty* `FBoneSocketTarget`, which is what you want for a pure
   component-space effector: the location is then read straight in component space. Because the verb
   makes the bone required, a caller who wants component-space IK has to pass some bone — `root` is
   the obvious choice — and the location is then resolved relative to that target instead, which is a
   different frame. Nothing warns about this.

## Cost in this project

Authoring a procedural crouch (no crouch clip ships with the UE5 Manny set) I called:

    add_two_bone_ik { ikBone: "foot_l", effectorBone: "root", jointTargetBone: "root",
                      effectorLocationSpace: "BCS_ComponentSpace", ... }

then set `effector_location` to the foot's measured reference-pose component-space position via
Python. The node compiled clean, `anim.decompile_agir` echoed the values back, and the pose was
wrong in play: the crouched character measured **153.8 cm** head-to-foot against a **141 cm**
standing baseline — 13 cm *taller*, pelvis raised rather than lowered. Nothing in the compile,
the decompile or the blackboard read showed a problem; only measuring bone sockets in PIE did.

## Ask

Add optional `effectorLocation` and `jointTargetLocation` (`{x,y,z}`) parameters, and let
`effectorBone` / `jointTargetBone` be omitted to leave the target empty for pure space-relative IK.
Failing that, document in the wiki page that a supplied target bone reinterprets the location and
that the locations are not settable through the verb.

## History
- `#2-still-open-in-fps-build` `OPEN` AI-stream - Direct route retried after the 2026-09-05 plugin pull; **still not available**. `animation.authoring.add_two_bone_ik {blueprintPath:'/Game/FPS/AI/ABP_Enemy', ikBone:'foot_l', effectorBone:'root', jointTargetBone:'root', effectorLocation:{x:20.55,y:11.02,z:8.48}, x:3200, y:800}` is refused with `UNKNOWN_PARAMS`: "Unknown parameter(s) ... [effectorLocation]. Valid parameters: [blueprintPath, graphName, ikBone, effectorBone, jointTargetBone, effectorLocationSpace, jointTargetLocationSpace, x, y, alpha, save]". The regenerated wiki page lists the same set and still marks `effectorBone`/`jointTargetBone` required, so neither ask in this ticket has landed. Nothing was mutated - the call is rejected before touching the graph. Workaround from the original report stands: create the node with the verb, then set `effector_location` / `joint_target_location` on the inner `FAnimNode_TwoBoneIK` in Python and write the struct back. New evidence for the fix's test matrix: clearing `effector_target` / `joint_target` `bone_reference.bone_name` to `None` **is** possible in Python and does change behaviour - with the targets bound to `root` the component-space effector was reinterpreted against that bone and a crouch pose came out 13 cm TALLER than standing (head-foot 153.8 crouched vs 141.1 standing, measured on live PIE sockets); with the targets cleared the node stops fighting the pose. That asymmetry between a settable location and an unsettable target is exactly why the verb needs both parameters. Returned to the workaround.

- `#3-locations-and-optional-targets` `IN-REVIEW` developer — Implemented both asks in `Handlers/Animation/AnimationAuthoringHandler_AnimBlueprint.cpp`. New `effectorLocation` / `jointTargetLocation` (`RPC_PARAM_OPT_NESTED` {x,y,z}, closed from first release, added to the `AdoptionSetIsRatcheted` list in `Tests/Infra/TestNestedParamKeyGate.cpp`): all three axes must be finite JSON numbers, else `INVALID_TRANSFORM_PAYLOAD`; extra keys → `UNKNOWN_NESTED_PARAMS`; applied values echoed in the response. `effectorBone` / `jointTargetBone` now optional (omitted/'' = empty `FBoneSocketTarget`), but refused with `MISSING_PARAMETERS` when their space is `BCS_BoneSpace`/`BCS_ParentBoneSpace`, because `FAnimNode_TwoBoneIK::IsValidToEvaluate` skips the node with an empty bone-relative target. **Correction to the report's diagnosis:** in UE 5.8 `FAnimationRuntime::ConvertBoneSpaceTransformToCS` ignores the target bone for `BCS_ComponentSpace` (and `BCS_WorldSpace` only uses the component transform), so a `root` target does NOT reinterpret a component-space effector; the likelier cause of the 13 cm taller crouch is the joint target left at component-space (0,0,0). Wiki now says this explicitly. Test `PinWright.animation.authoring.AddTwoBoneIkLocationsAndEmptyTargets` (real dispatcher; 4 refusals with codes + no node created; success authors locations with empty targets and echoes them). Docs: `docs/wiki-src/animation.authoring.md`, `CHANGELOG.md`. Fastcheck OK; not run in an editor yet.

- `#4-pin-defaults-and-correction` `IN-REVIEW` developer — Review found the `#3` location write was lost at compile: `EffectorLocation`, `JointTargetLocation` and `Alpha` are `PinShownByDefault`, `ReconstructNode` kept the pins' autogenerated defaults and the anim compiler copies unlinked exposed pin defaults over the struct, so the compiled node got `(0,0,0)` while the response echoed the request. Added `SyncSkeletalControlPinDefaultsFromStruct` (struct → pin via schema `TrySetDefaultValue`, then pin → struct so the echo is what compiles) after `ReconstructNode` in `add_two_bone_ik` and `add_modify_bone` (same defect for `translation`/`rotation`/`scale`/`alpha`). Tests: `PinWright.animation.authoring.AddTwoBoneIkLocationsAndEmptyTargets` now asserts the pin defaults (and gained a `jointTargetBone`+`BCS_ParentBoneSpace` refusal); new `PinWright.animation.authoring.AddModifyBoneWritesExposedPinDefaults`. **Correction to `#3`:** the joint-target explanation for the 13 cm taller crouch is unverified. The same pin mechanism is the more likely cause: the reporter set `effector_location` on the struct from Python, so the compiled node probably ran with an effector at the component origin. The target-bone claim in `#3` stands (component space never reads the target bone). Generic setters may share the defect: filed `B-anim-node-struct-writes-lost-to-exposed-pin-defaults`. Fastcheck OK; not run in an editor.
- `#5-verified-linux` `DONE` tester — Passed non-skipped in run3/full: `PinWright.animation.authoring.AddTwoBoneIkLocationsAndEmptyTargets`, `PinWright.animation.authoring.AddModifyBoneWritesExposedPinDefaults`, `PinWright.infra.dispatcher.NestedParamKeyGate.AdoptionSetIsRatcheted`. Acceptance met: optional `effectorLocation` / `jointTargetLocation` {x,y,z} are authored and land in the exposed pin defaults that compile, so the struct write is no longer lost at compile (#4). `effectorBone` / `jointTargetBone` may be omitted for an empty target; that is refused only for bone-relative spaces, where the node would not evaluate. Bad payloads are refused with no node created. Coverage limit: the reporter's PIE crouch measurement (153.8 vs 141 cm) was not repeated; #3/#4 dispute the target-bone diagnosis, and that disputed root cause is not demonstrated live.
