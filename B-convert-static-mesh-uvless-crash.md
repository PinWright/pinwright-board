---
id: B-convert-static-mesh-uvless-crash
title: "convert_to_static_mesh / convert_to_nanite / generate_lods HARD-CRASH the editor on a UV-less DynamicMesh (MikkT tangent OOB on the async build worker)"
status: IN-REVIEW
severity: Critical
category: bug
tags: [geometry, convert_to_static_mesh, convert_to_nanite, generate_lods, static-mesh, mikkt, tangents, uv, crash, editor-crash, mesh-bake, async-build-worker]
encounters: 1
lastSeen: 2026-07-04T11:52:31Z
---

# `geometry.convert_to_static_mesh` (and `convert_to_nanite` / `generate_lods`) crash the editor when the source DynamicMesh has no UV channel

`geometry.convert_to_static_mesh`, `geometry.convert_to_nanite`, and `geometry.generate_lods`
crash the editor with a fatal assert when the source DynamicMesh has no UV channel. A
DynamicMesh authored via `geometry.append_buffers` / `append_vertex` / `append_triangle`
WITHOUT `uvs` has no UV layer. The StaticMesh RENDER build (`FStaticMeshRenderData::Cache`
on the async build worker) runs MikkT tangent generation whenever `bRecomputeTangents` OR
`bRecomputeNormals` is set (both default true), and MikkT indexes a size-0 UV array → OOB
assert → the editor process dies (MCP connection drops, WinError 10054). The geometry-script
`FGeometryScriptCreateNewStaticMeshAssetOptions::bEnableRecomputeTangents` option does NOT
gate this render build (normal recompute alone still invokes MikkT), so toggling it does not
help. An RPC must never crash the editor on valid input.

Crash: `Assertion failed: (Index >= 0) & (Index < ArrayNum) [Array.h] "Array index out of
bounds: 0 into an array of size 0"`, callstack `FStaticMeshOperations::ComputeMikktTangents
[StaticMeshOperations.cpp:1866] <- ComputeTangentsAndNormals <- SetupRenderMeshDescription
<- FStaticMeshBuilder::Build <- FStaticMeshRenderData::Cache <- UStaticMesh::CacheDerivedData`
(Background Worker). Repro: create_procedural_mesh → append_buffers with 4 verts / 4 tris and
NO `uvs` → convert_to_static_mesh → editor crashes.

**Workaround:** none needed for a UV-having mesh (a create_box/sphere primitive carries UVs);
for a hand-authored UV-less mesh, add UVs first via `geometry.auto_uv` / `geometry.unwrap_uv`
(or pass `uvs` to `append_buffers`) before baking.
**Fix:** shared helper `GeometryUtils::EnsureMeshHasUVs(UDynamicMesh*)` adds a deterministic
`SetNumUVSets(1)` + `SetMeshUVsFromBoxProjection` UV set (never XAtlas) when `MeshHasUsableUVs`
is false, called before the bake at all three sites; `bEnableRecomputeNormals/Tangents` are
gated on the result as belt-and-suspenders for the degenerate/empty-mesh case.

## History
- `#1-initial-crash-repro` `OPEN` reporter — `geometry.convert_to_static_mesh` hard-crashes the editor on a UV-less DynamicMesh. Repro: (1) `geometry.create_procedural_mesh {name:"SmokeTetra", location:{x:-600,y:0,z:100}}`; (2) `geometry.append_buffers {actorName:"SmokeTetra", vertices:[[0,0,0],[100,0,0],[50,100,0],[50,50,100]], triangles:[[0,1,2],[0,3,1],[0,2,3],[1,3,2]]}` (NO uvs) → `{appendedVertices:4, appendedTriangles:4}`; (3) `geometry.convert_to_static_mesh {actorName:"SmokeTetra", assetPath:"/Game/SmokeMeshes/SM_SmokeTetra"}` → EDITOR CRASHED (`Assertion failed: (Index >= 0) & (Index < ArrayNum) [Array.h]`, callstack `FStaticMeshOperations::ComputeMikktTangents [StaticMeshOperations.cpp:1866] <- ComputeTangentsAndNormals <- SetupRenderMeshDescription <- FStaticMeshBuilder::Build <- FStaticMeshRenderData::Cache <- UStaticMesh::CacheDerivedData`, Background Worker; MCP drops with WinError 10054). Root cause: the render build's MikkT tangent pass indexes [0] into a size-0 UV array because the mesh has no UV layer; it fires whenever `bRecomputeTangents` OR `bRecomputeNormals` is set (both default true), so the geometry-script tangent option does not gate it. Same latent crash on `convert_to_nanite` and `generate_lods` (both bake through `CreateNewStaticMeshAssetFromMesh` / `UStaticMesh::Build`).
- `#2-ensure-uvs-before-bake` `IN-REVIEW` developer — Added shared helper `GeometryUtils::EnsureMeshHasUVs(UDynamicMesh*)` (GeometryUtils.cpp:223; declared GeometryUtils.h:116) that, when `MeshHasUsableUVs` is false (GeometryUtils.cpp:197 — checks the primary UV layer exists AND has >0 elements), runs `SetNumUVSets(Mesh,1)` then a deterministic `SetMeshUVsFromBoxProjection` framed on the mesh bounding box (clamped to a positive minimum so a flat/degenerate/empty mesh can't yield a zero-scale frame). Never XAtlas (which can hang on degenerate meshes). Called before the bake at all THREE sites: `convert_to_static_mesh` (MeshOpsHandler.cpp:407), `convert_to_nanite` (MeshOpsHandler.cpp:1861), `generate_lods` (LODCollisionHandler.cpp:256); `CreateOptions.bEnableRecomputeNormals/Tangents` are gated on the returned `bHasUVs` at each site as belt-and-suspenders for the pathological no-UV-produced case. Regression test `PinWright.geometry.convert_to_static_mesh.NoUVMeshDoesNotCrash` (new TestGeometryConvertNoUVMesh.cpp:97-98) builds a UV-less tetrahedron and bakes it: PASSES under CLI automation (no crash, asset saved, source mesh reports hasUVs:true afterward). Uncommitted; not yet tester-verified/committed — a tester must verify + commit before DONE.
