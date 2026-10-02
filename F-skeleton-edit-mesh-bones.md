---
id: F-skeleton-edit-mesh-bones
title: "Bone edits cannot reach a SkeletalMesh (ref skeleton and skin weights), and no verb creates a Skeleton from an existing SkeletalMesh"
status: DONE
severity: Medium
category: feature
tags: [skeleton, skeletal-mesh, bones, skeleton-modifier, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# Mesh-side bone editing and skeleton-from-mesh

`skeleton.add_bone` / `remove_bone` / `set_bone_parent` edit only the USkeleton asset through
`FReferenceSkeletonModifier` (`Source/PinWright/Private/Handlers/Animation/SkeletonHandler.cpp:780,
852, 912`); the SkeletalMesh's reference skeleton and skin weights are never touched, so adding an IK,
twist or weapon bone to an imported mesh is impossible. `skeleton.create_skeleton` makes a bare root
bone and takes no mesh (`SkeletonHandler.cpp:691`); `.pwskel` refuses a preview mesh that is not
already bound (`Source/PinWright/Private/PwSkel/PwSkelAssetCreate.cpp:858`).

Competitors ("Skeleton editing" row): ue-mcp `begin/edit/commit/cancel_skeleton_edit` on
`USkeletonModifier` via reflection, whole-batch validation with near-miss names and skinned-bone guards,
commit in a transaction (`AnimationHandlers_Skeleton.cpp:723-1669`); session state is in-memory and lost
on restart. ue-mcp `create_skeleton` uses `USkeletonFactory` with `TargetSkeletalMesh` and restores the
mesh on failure (`AnimationHandlers.cpp:1360-1552`). VibeUE uses `USkeletonModifier` too but reparents
by remove+re-add (to dodge the modal), which likely drops weights.

## Engine API

- `USkeletonModifier` (`SkeletalMeshModifiers` module, Editor): 5.8 `C:/UE_5.8/Engine/Plugins/Runtime/MeshModelingToolset/Source/SkeletalMeshModifiers/Public/SkeletonModifier.h:127-257` — `SetSkeletalMesh` `:134`, `CommitSkeletonToSkeletalMesh` `:141`, batch `AddBones/RemoveBones/RenameBones/ParentBones/SetBonesTransforms`. Path drift: 5.3-5.5 `Plugins/Experimental/MeshModelingToolsetExp/...`, 5.6+ `Plugins/Runtime/MeshModelingToolset/...` (checked locally).
- **Hazard:** when the edited hierarchy is incompatible with the assigned USkeleton, `PreCommitSkeleton` opens a modal `SCustomDialog` (`.../Private/SkeletonModifier.cpp:758-829`, `ShowModal` `:816`). The compatibility checker (`FReferenceSkeletonCompatibilityChecker`, parent-chain match plus bone-match ratio) is private.
- Skeleton from mesh: `USkeletonFactory::TargetSkeletalMesh` (`C:/UE_5.8/Engine/Source/Editor/UnrealEd/Classes/Factories/SkeletonFactory.h:24`; do not call `ConfigureProperties`, it opens a picker), or `USkeleton::MergeAllBonesToBoneTree` (`Skeleton.h:809`) + `Mesh->SetSkeleton`.

## Proposed scope

- `skeleton.edit_mesh_bones {skeletalMeshPath (required), ops[] (add|remove|rename|reparent|set_transform), dryRun}` — stateless: one batch is the session.
  - Validate the whole batch against a hierarchy model first (bone existence, cycles, name clashes, skinned bones on remove).
  - Replicate the engine compatibility check before commit and refuse `SKELETON_EDIT_INCOMPATIBLE` naming the offending bones, so the modal never opens.
  - Commit in an `FScopedTransaction`; read the mesh ref skeleton back and report the measured bone list, `skinnedBonesRemoved[]`, and save state.
  - Load the module conditionally or resolve by reflection (plugin path and name differ across 5.3-5.8).
- `skeleton.create_skeleton` gains optional `fromSkeletalMesh`; verify an exact hierarchy match and that the mesh now references the new skeleton. This part can ship first.

## Acceptance

- Adding a leaf bone to a mesh commits, reads back, and the mesh still renders.
- An incompatible reparent is refused with `SKELETON_EDIT_INCOMPATIBLE`; no modal appears under `-unattended`.
- `dryRun` writes nothing (dirty flags unchanged).
- `fromSkeletalMesh` produces an exact hierarchy match.

**Effort:** M-L (`fromSkeletalMesh` alone: S).

**Related:** `B-skeleton-bone-edits-desync-bone-tree` (fix first), `F-skeleton-no-mesh-for-physics-asset` (reverse direction).

## History
- `#1-no-mesh-bone-editing` `OPEN` reporter — Filed from the 2026-09-30 animation gap analysis at plugin HEAD `2580e7f4`. Severity Medium: hard blocker with only `python.execute` as a workaround, rare reach.
- `#2-edit-mesh-bones-and-from-mesh` `IN-REVIEW` developer — Both halves shipped; compile-checked (UBT `-SingleFile`, 5.8 Linux), not yet run in an editor. **New verb `skeleton.edit_mesh_bones`** (`Source/PinWright/Private/Handlers/Animation/SkeletonMeshBonesHandler.cpp`): `{skeletalMeshPath, ops[], dryRun}`, ops `add|remove|rename|reparent|set_transform` with `bone/parent/newName/location/rotation/scale/removeChildren` (nested keys declared). Drives the engine `USkeletonModifier` by reflection (`/Script/SkeletalMeshModifiers.SkeletonModifier`, every UFUNCTION + param name verified up front, `NOT_SUPPORTED` otherwise), so no link dependency on MeshModelingToolset(Exp). Each op is validated against the modifier's private hierarchy copy (`BONE_NOT_FOUND`/`BONE_EXISTS`/`PARENT_REQUIRED`/`PARENT_NOT_FOUND`/`PARENT_CYCLE`/`CANNOT_REMOVE_ROOT`, index named); then a port of the private `FReferenceSkeletonCompatibilityChecker` refuses the new code `SKELETON_EDIT_INCOMPATIBLE` (naming the bone; `bone`/`skeletonPath` in the payload) before `CommitSkeletonToSkeletalMesh` could reach its `SCustomDialog::ShowModal`. Commit runs in an `FScopedTransaction`; response reads `bones[]` back from the mesh, `skinnedBonesRemoved[]` (section `BoneMap` membership), `skeletonCompatible`, mesh save report. Engine content / engine-bound skeleton refused; multi-LOD meshes refuse index-moving batches (`OPERATION_NOT_SUPPORTED`) because the modifier remaps LOD 0 weights only. Did not reuse `EditSkeletonHierarchy`: the modifier updates the Skeleton itself through `MergeAllBonesToBoneTree` / `RecreateBoneTree` with retarget modes restored. **`skeleton.create_skeleton fromSkeletalMesh`** (`SkeletonHandler.cpp`): `MergeAllBonesToBoneTree(Mesh, false)` directly (the factory's failure path is a modal), bone-for-bone name+parent verification before rebinding, then `SetSkeleton` + preview mesh; `rootBoneName` with it is `INVALID_ARGUMENT`; `boneCount`/`rootBoneName` now read back (was a literal `1`). New code `ERR_SKELETON_EDIT_INCOMPATIBLE` in `Handlers/ErrorCodes.h`. Docs: `docs/wiki-src/skeleton.md` (`skeleton.create_skeleton` paragraph, new `### skeleton.edit_mesh_bones`), `docs/engine-version-support.md` (Checked-and-portable bullet). Tests (`Source/PinWright/Private/Tests/Animation/TestSkeletonEditMeshBones.cpp`, SkeletalCube copies under `/Game/PinWrightTests`): `PinWright.skeleton.edit_mesh_bones.AddLeafBoneReachesMeshAndSkeleton`, `.IncompatibleReparentIsRefused`, `.DryRunWritesNothing`, `PinWright.skeleton.create_skeleton.FromSkeletalMeshCopiesHierarchy`. Not covered by a test: rename / remove / set_transform ops and `skinnedBonesRemoved`.
- `#3-nested-gate-ratchet` `IN-REVIEW` developer — Full-suite run: all 4 new skeleton tests passed; `PinWright.infra.dispatcher.NestedParamKeyGate.AdoptionSetIsRatcheted` failed because `skeleton.edit_mesh_bones:ops` adopts the nested-key gate. Added it to `ExpectedAdopters` in `Tests/Infra/TestNestedParamKeyGate.cpp` and changed the `ops` description (handler + `docs/wiki-src/skeleton.md`) to say the eight keys are the whole op schema and any other key is refused `UNKNOWN_NESTED_PARAMS`. Both files pass the `-SingleFile` compile check.
- `#4-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). All four Acceptance tests passed in w23-final: `PinWright.skeleton.edit_mesh_bones.AddLeafBoneReachesMeshAndSkeleton` (commit, read back from mesh and skeleton, LOD0 render data still has vertices), `.IncompatibleReparentIsRefused` (`SKELETON_EDIT_INCOMPATIBLE` before the modal, under `-unattended`), `.DryRunWritesNothing`, and `PinWright.skeleton.create_skeleton.FromSkeletalMeshCopiesHierarchy` (exact hierarchy match, mesh rebound). Limits: the rename, remove and set_transform ops and `skinnedBonesRemoved` have no test; the 5.3-5.7 module path (reflection on `SkeletonModifier`) is unverified here.
