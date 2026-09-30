---
id: F-animation-audit-joint-limits
title: "No check that an animation's joint rotations stay within physical or learned limits"
status: OPEN
severity: Low
category: feature
tags: [animation, joint-limits, pose-validation, audit, deferred, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# Joint-limit audit for animation sequences

**Deferred (priority):** filed for tracking from the 2026-09-30 gap analysis; lower priority than the
other animation gaps. No blocking ticket or review date applies, so no machine gate is set.

An agent authoring clips (`.pwanim`, `set_bone_track_keys`) has no way to catch a hyperextended elbow
or a knee bending backwards. The only limit data PinWright touches is physics-asset constraints via
`skeleton.configure_constraint_limits` (`Source/PinWright/Private/Handlers/Animation/PhysicsAssetHandler.cpp:1061`).
`animation.measure_motion` is the precedent for a numeric gate over sequence data.

Competitor ("Learned bone rotation limits and pose validation" row): only VibeUE, and it is thin
(`Source/VibeUE/Private/PythonAPI/USkeletonService.cpp`): learned limits live in an in-memory static map
(`:1135`, lost on restart); skeleton matching is a fuzzy name "contains" (`:1340-1343`); per-axis Euler
min/max or P5/P95 with no ±180° wrap handling; `ValidateBoneRotation` passes silently when nothing was
learned; the hinge clamp is overwritten by the min/max clamp (`:1717`, `:1760`); `ValidatePose`
checks only pending preview deltas, not keyed animation (`UAnimSequenceService.cpp:3058`).

## Engine

No engine pose-validator API. Limit data exists in physics-asset constraints (swing1/swing2/twist),
IK Rig FBIK bone settings (`C:/UE_5.8/Engine/Plugins/Animation/IKRig/Source/IKRig/Public/Rig/Solvers/IKRigFullBodyIK.h:59-113`;
solvers became structs in 5.6), and `FAnimNode_ApplyLimits`
(`Source/Runtime/AnimGraphRuntime/Public/BoneControllers/AnimNode_ApplyLimits.h:15-35`).

## Proposed scope

Stateless audit `animation.audit_joint_limits {assetPath (required), limits: {physicsAsset} | {ranges} | {referenceSequences[]} (required, exactly one)}`:
- Swing-twist decomposition against the reference pose (maps directly onto physics constraint cones).
- `referenceSequences[]` computes ranges in the same call and returns them for reuse; no hidden cache.
- Per bone: `{maxSwingDeg, maxTwistDeg, limit, violationFrames}`; verdict through the audit framework (`docs/rpc-design.md` §18).
- Implement the physics-asset source first.

## Acceptance

- A `.pwanim` with an elbow past its limit is flagged with frames.
- The same clip within limits passes.
- Ranges computed from reference sequences are returned in the response.
- No package dirty-flag change.

**Effort:** M.

## History
- `#1-no-joint-limit-audit` `OPEN` reporter — Filed from the 2026-09-30 animation gap analysis at plugin HEAD `2580e7f4`; priority deferred. Severity Low: niche, and the one competitor implementation is a weak heuristic.
