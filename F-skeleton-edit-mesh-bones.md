---
id: F-skeleton-edit-mesh-bones
title: "Bone edits cannot reach a SkeletalMesh (ref skeleton and skin weights), and no verb creates a Skeleton from an existing SkeletalMesh"
status: OPEN
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
