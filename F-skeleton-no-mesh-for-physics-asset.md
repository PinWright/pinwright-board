---
id: F-skeleton-no-mesh-for-physics-asset
title: "No way to author a SkeletalMesh / preview mesh for a bare USkeleton, so create_physics_asset is unreachable from skeleton.create_skeleton"
status: IN-REVIEW
severity: Medium
category: feature
tags: [skeleton, skeletal-mesh, physics-asset, authoring, preview-mesh, ragdoll]
---

# No way to author a SkeletalMesh / preview mesh for a bare USkeleton

`skeleton.create_skeleton` produces a bare `USkeleton` with no bound
`USkeletalMesh` and no preview mesh. Every physics-asset authoring path in the
MCP is keyed on a `USkeletalMesh`, so once you have authored a skeleton purely
through `skeleton.*` (add_bone, create_socket, create_virtual_bone, etc.) there
is no RPC that lets you create a physics asset / ragdoll for it:

- `skeleton.create_physics_asset` requires `skeletalMeshPath`. The
  `skeletonPath` alias is documented to "resolve to the bound mesh", but a
  freshly authored bare skeleton has no bound mesh, so it returns
  `[MESH_NOT_FOUND] Failed to load skeletal mesh: /Game/Rigs/SK_HeroArm`.
- `physics.setup_physics_simulation` `skeletonPath` is documented to "fall back
  to preview mesh", but a bare skeleton has none, so it returns
  `[ASSET_NOT_FOUND] asset not found: skeleton /Game/Rigs/SK_HeroArm (no preview
  mesh for physics simulation)`.
- `skeleton.add_physics_body` requires an already-existing `physicsAssetPath` —
  chicken-and-egg, since the only way to get one is `create_physics_asset`,
  which needs a mesh.
- `skeleton.set_physics_asset` also requires a `skeletalMeshPath`.

There is **no** `skeleton.create_skeletal_mesh`, no `skeleton.set_preview_mesh`,
and no other RPC that binds a `USkeletalMesh` / preview mesh to a bare
`USkeleton` (confirmed by enumerating the full `skeleton.*` wiki namespace and
all `*mesh*` wiki pages). As a result the natural, documented workflow
"author a skeleton, then create a physics asset / ragdoll for it" is unreachable
end-to-end through the MCP from a `skeleton.create_skeleton`-authored skeleton.
The only escape is `python.execute`, which the realism constraints forbid as an
RPC substitute.

This is the missing counterpart to the existing skeleton-authoring family
(`create_skeleton` → `add_bone` → `create_socket` → `create_virtual_bone`):
that family lets you build a skeleton from scratch, but the physics half of the
toolset assumes the skeleton already came from an imported `USkeletalMesh`.

The errors themselves are clean, correct, well-formed (proper error codes +
descriptive text) — this is a capability gap, not a tool bug.

**What it should do:** add an RPC that gives a bare authored `USkeleton` a
`USkeletalMesh` / preview mesh so the physics-asset path becomes reachable.
Candidates:
- `skeleton.create_skeletal_mesh(skeletonPath, outputPath, ...)` — create a
  minimal `USkeletalMesh` bound to the skeleton (even a degenerate / single-bone
  render mesh would unblock physics-asset generation), and/or
- `skeleton.set_preview_mesh(skeletonPath, skeletalMeshPath)` — set
  `USkeleton::PreviewSkeletalMesh` so `physics.setup_physics_simulation`'s
  documented preview-mesh fallback works, and/or
- let `skeleton.create_physics_asset` accept a `skeletonPath` and build the
  physics asset directly from the skeleton's bone reference poses (it already
  documents auto-generating capsule bodies "from bone reference poses", which a
  bare skeleton has) instead of hard-requiring a `USkeletalMesh`.

**Workaround:** none via RPC for a bare authored skeleton; only `python.execute`.

## Repro
1. `skeleton.create_skeleton` `{path:/Game/Rigs/SK_HeroArm}` → ok,
   `boneCount:1`.
2. (optionally add bones/sockets/virtual bones — all succeed).
3. `skeleton.create_physics_asset` `{skeletalMeshPath:/Game/Rigs/SK_HeroArm}` →
   `[MESH_NOT_FOUND] Failed to load skeletal mesh: /Game/Rigs/SK_HeroArm`.
4. `physics.setup_physics_simulation`
   `{skeletonPath:/Game/Rigs/SK_HeroArm, assignToMesh:true}` →
   `[ASSET_NOT_FOUND] asset not found: skeleton /Game/Rigs/SK_HeroArm (no
   preview mesh for physics simulation)`.
5. No `skeleton.create_skeletal_mesh` / `skeleton.set_preview_mesh` exists in
   the `skeleton.*` namespace to bridge the two.

## History
- `#1-initial-repro` `OPEN` reporter — `skeleton.create_skeleton` yields a bare
  meshless `USkeleton`; `skeleton.create_physics_asset` (needs `skeletalMeshPath`)
  returns `[MESH_NOT_FOUND]` and `physics.setup_physics_simulation`
  (`skeletonPath` falls back to preview mesh) returns `[ASSET_NOT_FOUND] ... no
  preview mesh`. No `skeleton.create_skeletal_mesh` / `set_preview_mesh` exists,
  and `add_physics_body` / `set_physics_asset` both presuppose a mesh, so the
  documented "create a physics asset for this skeleton" workflow is unreachable
  from an MCP-authored skeleton. Replay-confirmed both errors verbatim on
  `/Game/Rigs/SK_HeroArm`. Errors are clean/correct — capability gap, not a bug.
- `#2-fix-create-physics-asset-from-bare-skeleton` `IN-REVIEW` developer —
  Implemented candidate 3: `skeleton.create_physics_asset` now accepts a bare
  authored `USkeleton` path. When the path doesn't resolve as a `USkeletalMesh`
  it falls back to loading it as a `USkeleton` and building the physics asset
  directly from the reference-pose bones — one capsule `USkeletalBodySetup` per
  bone, sized from each bone's farthest-child span in component space (mesh-free
  analogue of `UPhysicsAssetFactory`'s "capsule bodies from bone reference
  poses"); leaf bones get a default-sized capsule. The existing
  SkeletalMesh-keyed factory path is untouched (only reached when the path is a
  real mesh), so this is purely additive. Result now carries
  `fromSkeleton:true`, `skeletonPath`, and `bodyCount`. Added optional
  `minBoneLength` param (default 5 cm) to skip tiny joints, and updated the
  handler summary/param docs to advertise the skeleton path.
  Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Animation/PhysicsAssetHandler.cpp`
  (new `LoadSkeletonFromPathPhys` + `BuildBodiesFromSkeletonRefPose` helpers and
  the create_physics_asset bare-skeleton branch).
  Test: `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAnimationHandlers.cpp`
  `FPhysicsCreateAssetFromBareSkeletonTest`
  (`EditorAutomationRpcGateway.skeleton.create_physics_asset.FromBareSkeleton`) —
  drives the real production chain create_skeleton -> add_bone ->
  create_physics_asset through the registration table, asserts success (not
  `MESH_NOT_FOUND`), `bodyCount >= 1`, the asset loads, and a generated body
  carries a capsule primitive. Reverting the fallback makes the create call
  return `MESH_NOT_FOUND` and the test fails.
