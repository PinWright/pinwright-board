---
id: B-asset-dump-skeletal-mesh-summary-missing
title: "asset.dump produces no skeletal_mesh.json sidecar — SkeletalMesh LOD/material/collision data buried in escaped struct strings"
status: DONE
severity: High
category: bug
tags: [asset-dump, mesh, native-summary]
---

# asset.dump produces no skeletal_mesh.json sidecar — SkeletalMesh LOD/material/collision data buried in escaped struct strings

StaticMesh assets get a structured `static_mesh.json` (LOD count, triangles, vertices, materials, bounds, lightmap res, collision flag) per `F-asset-dump-native-summary-aspects`. SkeletalMesh assets get nothing equivalent — LOD info, triangle/vertex counts, material slots, and physics asset are buried inside escaped one-line struct strings in `properties.json`.

63 SkeletalMesh assets observed in slice; none have a typed sidecar.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Meshes/CargoCar/SK_CargoCart/`.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Meshes/Clutch/SK_Clutch02/`.
3. Observe: only `meta.json` + `properties.json`; no `skeletal_mesh.json`.

**Fix (proposed):** `AssetDumpHandler.cpp:618-625` only dispatches to `StaticMeshDumpBuilder` and `Texture2DDumpBuilder`. Add a `SkeletalMeshDumpBuilder` matching the StaticMesh shape (LOD counts, vertex/triangle counts, material slots, bounds, physics asset path).

## History
- `#1-initial-repro` `OPEN` reporter — No `skeletal_mesh.json` sidecar produced for any SkeletalMesh — the LOD/material/collision/physics-asset data only survives as escaped one-line struct strings in `properties.json`. Sample paths: `App/Meshes/CargoCar/SK_CargoCart/`, `App/Meshes/Clutch/SK_Clutch02/`. 63 SkeletalMesh assets in slice; zero typed summaries.
- `#2-add-skeletal-mesh-builder` `IN-REVIEW` developer — Added `SkeletalMeshDumpBuilder.{h,cpp}` mirroring the StaticMesh builder; routes via `AssetDumpHandler.cpp` dispatch; new `DumpFileNames::SkeletalMesh = "skeletal_mesh.json"` and canonical-list entry. Emits bounds, per-LOD triangle/vertex/section/tex-coord counts, materials, physics asset, skeleton. Added `FSkeletalMeshDumpBuilderEmitsTest`.
- `#3-verify-fix` `DONE` tester — Verified: ran `asset.dump` on `/App/Meshes/CargoCar/SK_CargoCart.SK_CargoCart`; response Files list now includes `skeletal_mesh.json` alongside `meta.json` + `properties.json`. File contains bounds (origin/extent/sphereRadius), lods=4, materials[] (2 slots w/ MI paths), numTexCoordsByLod [1,1,1,1], physicsAsset, sectionsByLod [2,2,2,2], skeleton, trianglesByLod [18649,4661,2331,1118], verticesByLod [18027,5956,3565,2069].
