---
id: F-rpc-mesh-describe-skeletal
title: "Add live RPC describing SkeletalMesh asset (bounds, materials, LODs, links)"
status: DONE
severity: Medium
category: feature
tags: [asset-dump, skeletal-mesh, rpc-coverage]
---

# Add live RPC describing SkeletalMesh asset (bounds, materials, LODs, links)

`SkeletalMeshDumpBuilder::BuildSkeletalMeshJson` emits a `skeletal_mesh.json` sidecar with `bounds.{origin,extent,sphereRadius}`, per-slot materials (`slot`, `path`, `importedSlot`), `lods` count, per-LOD `trianglesByLod` / `verticesByLod` / `sectionsByLod` / `numTexCoordsByLod`, plus `physicsAsset` and `skeleton` asset paths. None of this is reachable through a live RPC: the only `USkeletalMesh`-touching handlers in `SkeletalMeshHandler.cpp` are mutating ops (skin-weight normalize/prune/set/auto/copy/mirror, cloth bind/assign), and `skeleton.get_info` only summarises the bound `USkeleton` (bone/socket/virtual-bone counts), not the mesh's geometry/material/LOD state. Grep for `REGISTER_RPC_HANDLER\("(skeletal_mesh|skeleton|mesh)\.` confirms there is no `mesh.*` namespace and no `skeleton.describe_mesh` / `skeleton.get_mesh_info`.

This violates the asset-dump-vs-RPC parity policy: anything an asset dump emits should also be reachable via a live RPC so callers don't have to round-trip through `asset.dump_folder` to read static mesh metadata.

**Fix:** Add a read-only handler that returns the same JSON shape as `SkeletalMeshDumpBuilder::BuildSkeletalMeshJson`. Implementation should call straight into `SkeletalMeshDumpBuilder::BuildSkeletalMeshJson(Mesh)` so the dump and the live RPC stay structurally identical. Recommended namespace `skeleton.describe_mesh` (or `skeleton.get_mesh_info`) — consistent with the existing `skeleton.*` neighborhood that already hosts mesh-touching ops in `SkeletalMeshHandler.cpp`. Required param `skeletalMeshPath` (string, asset path to `USkeletalMesh`); error code `MESH_NOT_FOUND` on load failure, matching the local SkeletalMeshHandler convention. Update `Docs/wiki/skeleton.md` with an H3 overlay for the new method.

## History
- `#1-initial-repro` `OPEN` reporter — `SkeletalMeshDumpBuilder` writes `skeletal_mesh.json` with bounds/materials/per-LOD geometry stats/physicsAsset/skeleton refs, but no live RPC exposes any of it. `skeleton.get_info` only reports skeleton bone/socket counts; all `SkeletalMeshHandler.cpp` RPCs are mutating skin-weight/cloth ops; no `mesh.*` namespace exists. Violates dump-vs-RPC parity. Suggested handler: `skeleton.describe_mesh(skeletalMeshPath)` delegating to `SkeletalMeshDumpBuilder::BuildSkeletalMeshJson`.
- `#2-describe-mesh-rpc` `IN-REVIEW` developer — Reformulated inaccurate fix prose to remove the nonexistent `skeleton.get_physics_asset_info` dependency and use the local `MESH_NOT_FOUND` convention. Added `skeleton.describe_mesh` in `SkeletalMeshHandler.cpp`, documented it in `docs/wiki/skeleton.md`, and added `FSkeletonDescribeMeshReturnsDumpShapeTest` to assert the live RPC returns the dump-builder field shape.
- `#3-verify-fix` `DONE` tester — Verified: `skeleton.describe_mesh` on `/App/Meshes/Truck/SK_Truck_Cockpit.SK_Truck_Cockpit` returned full dump-shape JSON with `bounds.{origin,extent,sphereRadius}`, 10 material entries with `slot`/`path`/`importedSlot`, `lods:1`, `trianglesByLod:[97911]`, `verticesByLod:[87294]`, `sectionsByLod:[10]`, `numTexCoordsByLod:[3]`, plus `physicsAsset` and `skeleton` asset paths. Wiki page also reachable via `call("skeleton.describe_mesh")`.
