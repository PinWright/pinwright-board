---
id: B-skeletal-mesh-missing-sockets-vbones
title: "skeletal_mesh.json missing sockets and virtual bones"
status: DONE
severity: Medium
category: bug
tags: [skeletal-mesh, sidecar, completeness]
---

# skeletal_mesh.json missing sockets and virtual bones

`skeletal_mesh.json` captures bounds, materials, LODs, and physics asset reference, but does NOT include the mesh's attachment sockets or virtual bones (which live on the parent skeleton).

The sibling `skeleton.json` correctly dumps sockets and virtualBones, but consumers inspecting a skeletal mesh shouldn't have to navigate to the skeleton separately for attachment-point queries.

## Sample

`App/App/Drone/PioneerGoPro/drone_GoPro/skeletal_mesh.json` — has bounds/materials/LODs; no sockets or virtual bones.

Verified in `SkeletalMeshDumpBuilder.cpp` — no iteration over `Mesh->GetSkeleton()->GetVirtualBones()` or sockets.

## Fix sketch

In `SkeletalMeshDumpBuilder.cpp`, emit root-level `sockets` from `USkeletalMesh::GetActiveSocketList()` so mesh-only sockets are included and skeleton duplicates are suppressed by Unreal's active-socket logic. Emit root-level `virtualBones` from `Mesh->GetSkeleton()->GetVirtualBones()` when a skeleton exists. Reuse the `SkeletonDumpBuilder` socket and virtual-bone JSON shape and emit empty arrays when absent. Bump `skeletal_mesh.json` aspect version.

## History
- `#1-skelmesh-sockets-vbones` `OPEN` reporter — separate from B-asset-dump-skeletal-mesh-summary-missing (which is about the sidecar not existing for some assets); this is about completeness of an existing sidecar.
- `#2-active-sockets-vbones` `IN-REVIEW` developer — Rewrote the fix sketch to use USkeletalMesh::GetActiveSocketList() instead of skeleton-only sockets; emitted skeletal_mesh.json sockets and virtualBones, bumped the skeletal_mesh.json aspect version, updated live-read docs, and added FSkeletalMeshDumpBuilderEmitsActiveSocketsAndVirtualBonesTest.
- `#3-verify-fix` `DONE` tester — Verified: `asset.dump` on `/Game/Characters/Heroes/Mannequin_UE4/Meshes/SK_Mannequin` produced `skeletal_mesh.json` with `sockets` containing 3 populated entries (Weapon_R, Weapon_L, Weapon_R_Muzzle with bone/location/rotation/scale) and `virtualBones: []`. Also confirmed empty-case via `/App/App/Drone/PioneerGoPro/drone_GoPro` (both keys present as empty arrays, matching skeleton sidecar).
