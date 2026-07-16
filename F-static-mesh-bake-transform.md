---
id: F-static-mesh-bake-transform
title: "static_mesh.bake_transform — bake rotation/translation/uniform scale into a StaticMesh asset in place (render LODs + simple collision + sockets + bounds)"
status: IN-REVIEW
severity: Medium
category: feature-request
tags: [static-mesh, bake-transform, convex-collision, mesh-description, geometry, asset-mutation]
encounters: 1
lastSeen: 2026-07-16T13:45:27+03:00
---

# `static_mesh.bake_transform` — transform a StaticMesh asset in place

There is no way to apply a transform to an existing StaticMesh asset through PinWright or UE Python. The `geometry.*` namespace is a DynamicMeshActor pipeline with no StaticMesh ingest (`convert_to_static_mesh` creates a NEW asset); `static_mesh.describe` is read-only; `asset.generate_lods` / `asset.nanite_rebuild_mesh` are the only StaticMesh mutators and neither touches geometry placement.

## The Python wall (why this must be a C++ RPC)

Hit live on 2026-07-16 while flipping `/App/App/Drone/SM_PioneerCircle` (artist mesh authored facing −X; the game needs +X forward, and a root component cannot be rotated relative to itself, so the fix must be baked into the asset):

- Render geometry WAS doable via `python.execute` + GeometryScript (`CopyMeshFromStaticMesh` → `TransformMesh` → `CopyMeshToStaticMesh`) — but only after enabling the GeometryScripting plugin per-session and generating wrappers via `get_type_from_class` (plugin classes registered after python's type scan).
- Simple collision was NOT doable. Verbatim python errors:
  - `Property 'Transform' for attribute 'transform' on 'KConvexElem' is protected and cannot be read`
  - `Failed to find property 'vertex_data' for attribute 'vertex_data' on 'KConvexElem'`
  `FKConvexElem` exposes nothing to Python beyond `export_text`/`to_tuple`. The only scripted fallback was `StaticMeshEditorSubsystem.set_convex_decomposition_collisions` — i.e. REGENERATING collision, which replaced the mesh's 19 authored hulls with a 15-hull approximation. Shape-changing fallback for what should be an exact rigid transform.

## Proposed API

`static_mesh.bake_transform { assetPath (req), rotation {pitch,yaw,roll}, translation {x,y,z}, scale (uniform number, default 1), save (default true) }`

In place, in one go:
1. All valid source-model LODs via `FStaticMeshOperations::ApplyTransform` (positions + normals/tangents, renormalized) + `CommitMeshDescription`; reduction-only LODs regenerate on build.
2. HiRes/Nanite source mesh description when present (otherwise Nanite rebuild reverts the render mesh).
3. Simple collision AggGeom exactly: convex `VertexData` transformed (after `BakeTransformToVerts()` when an elem carries a non-identity transform) + `UpdateElemBox()`; box/sphyl/tapered-capsule centers+rotations composed, extents scaled; spheres center+radius; level-set-style elems skipped and counted.
4. Sockets (RelativeLocation/Rotation/Scale).
5. Rebuild via `PreEditChange(nullptr)` … `PostEditChange()` (which runs Build internally) + `FStaticMeshCompilingManager::FinishCompilation` for a deterministic response; bounds extensions scaled (not rotatable — noted in response when non-zero).

Scope: rotation + translation + uniform positive scale only; mirror/shear/non-uniform scale rejected with `INVALID_ARGUMENT` (winding/normal complications, no current need). Response reports per-category transformed counts, new bounds, and the standard save report.

## History
- `#1-initial-report` `OPEN` reporter — Filed from the SM_PioneerCircle front/back flip: render geometry flipped via session-enabled GeometryScript python, but FKConvexElem's python-opaque data forced regenerating 19 authored hulls into a 15-hull approximation. An exact in-place bake needs C++; proposed `static_mesh.bake_transform` as above.
- `#2-implemented` `IN-REVIEW` developer — Implemented as proposed in `Source/PinWright/Private/Handlers/Asset/StaticMeshBakeTransformHandler.cpp` (plugin commit 3583b2c9): validates uniform scale > 0 (mirror/shear rejected INVALID_ARGUMENT), no-ops on identity, `PreEditChange(nullptr)` → per-LOD `FStaticMeshOperations::ApplyTransform` + `CommitMeshDescription` (reduction-only LODs counted as generatedLodsRebuilt), hi-res source transformed when present, AggGeom transformed exactly (convex: `BakeTransformToVerts()` when elem transform non-identity, then VertexData transform + `UpdateElemBox()`; box/sphere/sphyl/tapered-capsule composed via FQuat right-first order; level-set families skipped+counted, MLLevelSet/SkinnedTriangleMesh arrays guarded `UE_VERSION_NEWER_THAN_OR_EQUAL(5,6,0)` — they only exist from 5.6; all load-bearing APIs verified present 5.3–5.8), sockets transformed, bounds extensions scaled with response note, rebuild via single `PostEditChange()` + `FStaticMeshCompilingManager::FinishCompilation`, save via `SaveAssetToDiskReportingPresence` + honest save report. Wiki: `### static_mesh.bake_transform` section in `docs/wiki-src/static_mesh.md`. Regression tests `PinWright.static_mesh.bake_transform.Yaw180TransformsGeometryCollisionAndSockets` (one-triangle fixture + convex/box/socket, asserts (x,y)→(−x,−y) on MD verts, hull verts, ElemBox, box center/yaw, socket, response counts, save:false honored) and `.RejectsNonPositiveScale` (scale 0 and −1 → INVALID_ARGUMENT) — both green on UE 5.7, full `PinWright.static_mesh` bucket green. Live-verified on the motivating asset: original `/App/App/Drone/SM_PioneerCircle` restored from git and baked yaw-180 — response reported all 19 convex hulls transformed, face renders toward +X.
