---
id: B-static-mesh-missing-collision-body-counts
title: "static_mesh.json missing collision-body counts (sphere/box/convex/sphyl)"
status: DONE
severity: Low
category: bug
tags: [static-mesh, sidecar]
---

# static_mesh.json missing collision-body counts

`static_mesh.json` captures bounds, LODs, materials, lightmapResolution, and `collisionTraceFlag`, but omits the actual physics-body element counts that `physics_asset.json` provides for skeletal meshes (sphere/box/convex/sphyl element counts from `UBodySetup::AggGeom`).

## Sample

`App/App/Drone/drone_body/static_mesh.json` — only `collisionTraceFlag`, no body element counts.

## Fix sketch

`StaticMeshDumpBuilder.cpp` — after `collisionTraceFlag` emission, walk `Mesh->GetBodySetup()->AggGeom`:

```json
"collision": {
    "trace": "...",
    "elements": { "sphere": N, "box": N, "convex": N, "sphyl": N }
}
```

Mirror the physics_asset.json structure for consistency.

## History
- `#1-staticmesh-collision-counts` `OPEN` reporter — schema parity with skeletal mesh / physics asset dumps.
- `#2-staticmesh-collision-elements` `IN-REVIEW` developer - Added `static_mesh.json` nested `collision.elements` counts for sphere, box, sphyl, convex, and taperedCapsule while preserving the top-level `collisionTraceFlag`.
- `#3-verify-collision-elements` `DONE` tester — Verified: `asset.dump` on `/App/App/Drone/drone_body` produced `static_mesh.json` with `collision.elements` = { box: 2, convex: 0, sphere: 0, sphyl: 0, taperedCapsule: 0 } and preserved `collisionTraceFlag: "CTF_UseDefault"`.
