---
id: B-geometry-uv-gen-silent-noop
title: "geometry.project_uv / unwrap_uv / auto_uv / pack_uv_islands report success but create ZERO UV elements when the target UV layer doesn't already exist (silent no-op on a hand-authored mesh)"
status: OPEN
severity: High
category: bug
tags: [uv-gen-silent-noop, geometry, project_uv, unwrap_uv, auto_uv, pack_uv_islands, uv, success-no-effect, append_buffers]
encounters: 1
lastSeen: 2026-07-10T21:10:50.8332607+03:00
---

# UV-generation verbs silently no-op (success:true, 0 UV elements) on a mesh whose UV layer was never enabled

The whole `geometry` UV-generation family — `project_uv` (box/planar/cylindrical),
`unwrap_uv` (XAtlas), its alias `auto_uv`, and `pack_uv_islands` (same XAtlas
routine) — writes into `UVChannel` **without first ensuring that UV layer
exists**. On any DynamicMesh whose UV overlay for that channel is absent — the
normal state of a mesh hand-authored via `geometry.append_buffers` /
`append_vertex` / `append_triangle` without a `uvs` array — the underlying
GeometryScript call finds no layer at `UVChannel`, writes its complaint to the
`nullptr` debug object (discarded), and returns having done nothing. The handler
never checks that result and unconditionally reports `success:true`
("UV projection applied" / "UV unwrapping completed"). The mesh gains **zero** UV
elements: `geometry.get_mesh_info` still reports `hasUVs:false`, and a follow-up
`geometry.set_uvs` then fails `[NO_UV_ELEMENTS] No UV elements found for vertex N`.

The caller is told the UV op succeeded when it silently did nothing — a false
success on a normal authoring path.

## Affected methods (same root cause — one fix)

- `geometry.project_uv` — box / planar / cylindrical projection. Silent no-op (confirmed by replay).
- `geometry.unwrap_uv` — XAtlas auto-unwrap. Silent no-op (confirmed by replay).
- `geometry.auto_uv` — alias of `unwrap_uv`, shares the same `ApplyXAtlasUnwrap` routine → same defect.
- `geometry.pack_uv_islands` — also routes through `ApplyXAtlasUnwrap` → same defect.

## Root cause (guilty source, read from plugin source — ground truth)

The maintainers already KNOW these GeometryScript calls need the UV layer to
pre-exist — the crash-fix helper does it explicitly, but the UV-generation
handlers do not.

Proof the layer must exist first — `GeometryUtils::EnsureMeshHasUVs`
(`Source/PinWright/Private/Handlers/Geometry/GeometryUtils.cpp:236-238`):

```cpp
// Guarantee UV channel 0 exists to project into (enables mesh attributes + a UV
// layer; the layer is still element-less until the projection fills it).
UGeometryScriptLibrary_MeshUVFunctions::SetNumUVSets(Mesh, 1, nullptr);
```

`geometry.project_uv` skips that guarantee —
`Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp:1682-1683`:

```cpp
UGeometryScriptLibrary_MeshUVFunctions::SetMeshUVsFromBoxProjection(
    Target.Mesh, UVChannel, ProjectionTransform, FGeometryScriptMeshSelection(), 2, nullptr);
```

then unconditionally succeeds — `MeshOpsHandler.cpp:1708`:
`Ctx.SendSuccess(TEXT("UV projection applied"), Result);` (the planar/cylindrical
branches at :1687 and :1692 have the identical shape).

`geometry.unwrap_uv` / `auto_uv` / `pack_uv_islands` skip it too — the shared
`GeometryUtils::ApplyXAtlasUnwrap` (`GeometryUtils.cpp:159-171`):

```cpp
UGeometryScriptLibrary_MeshUVFunctions::AutoGenerateXAtlasMeshUVs(
    Mesh, UVChannel, FGeometryScriptXAtlasOptions(), nullptr);
...
Ctx.SendSuccess(SuccessMsg, Result);
```

In every case the GeometryScript `Debug` argument is `nullptr` and the return is
never inspected, so a no-op (target UV layer missing) is indistinguishable from a
real unwrap and is reported as success.

## Fix

Before writing into `UVChannel`, ensure that layer exists (mirror
`EnsureMeshHasUVs`: `SetNumUVSets` / grow `SetNumUVLayers` up to `UVChannel`, as
`geometry.set_uvs` already does at `MeshInfoHandler.cpp:565-571`), OR pass a real
`FGeometryScriptDebug` and surface the "UVSetIndex does not exist" failure as an
error instead of `SendSuccess`. Apply at both call sites — `project_uv`
(MeshOpsHandler.cpp) and the shared `ApplyXAtlasUnwrap` (GeometryUtils.cpp) — so
all four verbs are covered.

## Verbatim repro (replayed at HEAD via mcp__pinwright__call)

1. `geometry.create_procedural_mesh {name:"ReplayPyramid", location:[0,0,0], enableCollision:true}` -> ok
2. `geometry.append_buffers {actorName:"ReplayPyramid", vertices:[[-50,-50,0],[50,-50,0],[50,50,0],[-50,50,0],[0,0,100]], triangles:[[0,2,1],[0,3,2],[0,1,4],[1,2,4],[2,3,4],[3,0,4]]}` (NO uvs) -> `{appendedVertices:5, appendedTriangles:6}`
3. `geometry.project_uv {actorName:"ReplayPyramid", projectionType:"box", scale:1}` -> `success:true` `{message:"UV projection applied"}`
4. `geometry.get_mesh_info {actorName:"ReplayPyramid"}` -> `hasUVs:false` (project_uv did nothing)
5. `geometry.unwrap_uv {actorName:"ReplayPyramid", uvChannel:0}` -> `success:true` `{message:"UV unwrapping completed"}`
6. `geometry.get_mesh_info {actorName:"ReplayPyramid"}` -> `hasUVs:false` (unwrap_uv also did nothing)

Result: two UV-generation verbs each reported success; the mesh still has zero UV
elements after both. (A subsequent `geometry.set_uvs {vertexIndex:0}` on such a
mesh returns `[NO_UV_ELEMENTS] No UV elements found for vertex 0` — the honest
downstream symptom.)

## Severity

severity rationale: impact=silent-false-success (caller trusts "UV projection
applied" / "UV unwrapping completed" while nothing happened) x reach=normal
(hand-authoring a mesh via append_buffers and then UV-ing it is a documented,
ordinary workflow; primitives carry UVs so the trap only bites hand-built meshes)
-> High.

## History
- `#1-initial-repro` `OPEN` reporter — Family-level silent no-op confirmed by direct replay at HEAD. `geometry.project_uv` (box) and `geometry.unwrap_uv` (XAtlas) both returned `success:true` on a 5-vertex/6-triangle append_buffers pyramid built without `uvs`, yet `geometry.get_mesh_info` reported `hasUVs:false` after each; `geometry.set_uvs` then errored `[NO_UV_ELEMENTS]`. Root cause: `project_uv` (MeshOpsHandler.cpp:1682-1683) and the shared `ApplyXAtlasUnwrap` (GeometryUtils.cpp:159-160 — used by `unwrap_uv`/`auto_uv`/`pack_uv_islands`) write into `UVChannel` without ensuring that UV layer exists (no `SetNumUVSets`/`SetNumUVLayers`), pass `nullptr` for the GeometryScript debug, and never check the result, so a missing-layer no-op is reported as success — while `GeometryUtils::EnsureMeshHasUVs` (GeometryUtils.cpp:236-238) and `geometry.set_uvs` (MeshInfoHandler.cpp:565-571) both DO enable the layer first, proving the requirement is known. Distinct from `B-convert-static-mesh-uvless-crash` (that's the convert/bake MikkT crash on a UV-less mesh — different code path and symptom; its EnsureMeshHasUVs fix is why convert now injects a box UV and the mesh only gains hasUVs after conversion) and from `E-geometry-auto-uv-redundant-with-unwrap-uv` (auto_uv/unwrap_uv naming/discovery duplication, not a silent no-op).
