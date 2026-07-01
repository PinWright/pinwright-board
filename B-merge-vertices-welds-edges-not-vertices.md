---
id: B-merge-vertices-welds-edges-not-vertices
title: "geometry.merge_vertices doesn't merge coincident vertices — it calls WeldMeshEdges (boundary-edge weld, a duplicate of geometry.weld_vertices), so it silently no-ops (merged:0) on proximity-coincident interior verts while reporting 'Vertices merged'"
status: IN-REVIEW
severity: Medium
category: bug
tags: [geometry, merge_vertices, weld, WeldMeshEdges, dynamic-mesh, silent-no-op, mesh-cleanup, semantic-mismatch]
---

# `geometry.merge_vertices` welds boundary EDGES, not coincident VERTICES — silently does nothing on the documented use case

`geometry.merge_vertices` is documented (wiki + handler summary) as **"Merge
nearby vertices on a dynamic mesh"** with a `tolerance` param described as
**"Merge distance tolerance"**. The implementation does NOT merge nearby
vertices. It calls `UGeometryScriptLibrary_MeshRepairFunctions::WeldMeshEdges`
(the `FMergeCoincidentMeshEdges` operation), which welds **open boundary edges**
whose endpoints are within tolerance — it never welds proximity-coincident
**interior** vertices on a closed mesh. On a watertight mesh (a sphere, box,
etc.) there are no open boundary edges, so the call is a no-op: it returns
`merged:0, verticesAfter:<unchanged>` while still reporting success
(`"message":"Vertices merged"`).

This is the canonical mesh-cleanup workflow — collapse/snap a few vertices onto
the same coordinate via `set_vertex_position`, then `merge_vertices` to weld the
now-coincident duplicates so downstream `remove_degenerates` can strip the
zero-area triangles. That workflow is impossible with the current
implementation: the documented "merge nearby vertices" verb provably leaves the
duplicates in place.

The correct primitive for "merge nearby vertices" is a coincident-**vertex**
weld, not the coincident-**edge** weld. There is no one-line GeometryScript
library swap for this: UE 5.7 exposes **no** `FMergeCoincidentMeshVertices` (only
`FMergeCoincidentMeshEdges`, which is what `WeldMeshEdges` already wraps), and the
GeometryScript `MergeMeshVerticesInSelections`
(`MeshBasicEditFunctions.h`) is a two-selection keep/discard merge that defaults
`bOnlyBoundary=true` — it would skip exactly the interior verts this verb needs.
The real fix is a lower-level `FDynamicMesh3` proximity-weld pass
(`FDynamicMesh3::MergeVertices` over a tolerance-clustered match, the same
primitive `MergeMeshVerticesInSelections` uses internally), guarded against the
non-manifold merges the engine refuses, with an honest merged-count.

Note the engine still **refuses** merges that would create non-manifold edges
(`FDynamicMesh3::MergeVertices` returns a non-`Ok` `EMeshResult`). So collapsing
3 strictly-interior coincident verts on a *fully closed watertight* sphere (the
literal repro below) can legitimately stay `merged:0` — but the verb must then
report the honest count and not silently claim success on a mesh where boundary /
open-region coincident verts *are* weldable. The fix is the real vertex-weld pass
plus an accurate count; the watertight-interior case is a known engine
limitation, not a bug in the verb once it reports truthfully.

## Verbatim repro (replayed via mcp__editor-automation__call)

1. `geometry.create_sphere {name:"PropCleanupSphere", radius:60, subdivisions:12, location:{x:0,y:0,z:0}}` → created (DynamicMeshActor).
2. `geometry.get_mesh_info {actorName:"PropCleanupSphere"}` → `vertexCount:728, triangleCount:1452`.
3. `geometry.get_vertex_position {actorName:"PropCleanupSphere", vertexIndex:0}` → `position:{x:-34.64101615137754, y:-34.64101615137754, z:-34.64101615137754}`.
4. `geometry.set_vertex_position {actorName:"PropCleanupSphere", vertexIndex:1, position:{x:-34.64101615137754,...}}` → set (vertex 1 now coincident with vertex 0).
5. `geometry.set_vertex_position {actorName:"PropCleanupSphere", vertexIndex:2, position:{x:-34.64101615137754,...}}` → set (vertex 2 now coincident too — 3 verts share one coord).
6. `geometry.get_mesh_info` → still `728/1452` (collapse made triangles degenerate, didn't delete them — expected).
7. **`geometry.merge_vertices {actorName:"PropCleanupSphere", tolerance:0.01}`** → `{"verticesBefore":728,"verticesAfter":728,"merged":0,"message":"Vertices merged"}` — **the 3 coincident verts were NOT welded** despite a tolerance (0.01) far exceeding their 0.0 separation.
8. `geometry.get_mesh_info` → `728/1452` (unchanged).

A tolerance of 0.01 against vertices at literally the same coordinate must weld
them if the verb does what it says. `merged:0` is the bug.

## Related secondary gap (same task, recorded here, not split)

In the same cleanup attempt, `geometry.remove_degenerates {actorName:"PropCleanupSphere"}`
returned `{"actorName":"PropCleanupSphere","message":"Degenerate geometry removed"}`
— **no removed-triangle/vertex count in the result**, so a caller can't confirm
whether anything was actually stripped (mesh stayed `728/1452`). That count-echo
omission is a low-severity ergonomic sibling of `E-geometry-deformer-echo-mesh-counts`
(mutators that don't echo post-op `vertexCount`/`triangleCount`); it is noted
here for context but the load-bearing defect is the `merge_vertices` semantics
above.

**Workaround:** none from the MCP surface for welding coincident interior verts
— `merge_vertices` is the only weld verb and it targets boundary edges. An agent
must restructure the mesh so the duplicates lie on an open boundary (rarely
possible), or do the cleanup outside the tool.

**Fix:** retarget `merge_vertices` at a real proximity coincident-**vertex** weld
over the underlying `FDynamicMesh3` (`UDynamicMesh::EditMesh` →
`FDynamicMesh3::MergeVertices` on each tolerance-matched pair, counting only
`EMeshResult::Ok` merges), so it welds proximity-coincident vertices within
`tolerance` as documented — then the collapse→merge→remove_degenerates cleanup
pass works for any weldable (non-manifold-blocked) duplicates. Report the honest
successful-merge count. Do **not** add a `geometry.weld_edges` verb: boundary-edge
welding already ships as `geometry.weld_vertices`
(`MeshOpsHandler.cpp:1172`), which wraps the *same*
`WeldMeshEdges` call `merge_vertices` was incorrectly using — the two were
duplicate boundary-edge welds; this fix makes `merge_vertices` an actual vertex
weld and leaves `weld_vertices` as the edge weld. Implementation:
`Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp` — handler at
~1300-1338, the wrong `WeldMeshEdges` call is line ~1320.

## History
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded then fixed. REWORD because the original Fix named a non-existent UE 5.7 API (`FMergeCoincidentMeshVertices` — only `FMergeCoincidentMeshEdges` exists), cited stale line numbers (`MeshOpsHandler.cpp:1258-1296`/call at 1278; actual handler ~1300-1338, wrong `WeldMeshEdges` call at ~1320, file now `Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp`), and proposed a new `geometry.weld_edges` verb that already effectively ships as `geometry.weld_vertices` (`MeshOpsHandler.cpp:1172`, the SAME `WeldMeshEdges` call). FIX: retargeted `geometry.merge_vertices` at a real proximity coincident-VERTEX weld — `UDynamicMesh::EditMesh` → `FDynamicMesh3::MergeVertices` over a tolerance-matched pairing of all vertices, counting only `EMeshResult::Ok` merges (so edge-connected coincident verts collapse via edge-collapse even on a closed mesh; non-manifold-blocked merges are honestly not counted instead of silently claiming success). Reports the accurate `merged` count and now echoes post-op `vertexCount`/`triangleCount` via `GeometryUtils::SetMeshCountFields`. Left `geometry.weld_vertices` as the boundary-edge weld (no new verb). Files: `Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp`. Test: `Source/PinWright/Private/Tests/Geometry/TestGeometryMergeVerticesWeldsVertices.cpp` (`PinWright.geometry.merge_vertices.WeldsCoincidentVertices`) — spawns a real box-sphere via the dispatcher, collapses a contiguous block of edge-adjacent low-index verts onto vertex 0's coord, runs `merge_vertices`, and asserts `merged>=1`, `verticesAfter<verticesBefore`, and a non-zero echoed `vertexCount`. Counterfactual: reverting to the `WeldMeshEdges` boundary-edge weld no-ops on the closed sphere (`merged:0`), failing the test. Did not compile/run (later phase). The `remove_degenerates` count-echo secondary gap stays owned by E-geometry-deformer-echo-mesh-counts (#8, IN-REVIEW) — not re-filed here.
- `#1-initial-repro` `OPEN` reporter — Filed from a mesh-cleanup task (collapse verts via set_vertex_position → merge_vertices → remove_degenerates). Replay-confirmed via mcp__editor-automation__call: created PropCleanupSphere (728v/1452t), moved vertexIndex 1 and 2 onto vertexIndex 0's coord, then `merge_vertices {tolerance:0.01}` returned `verticesBefore:728, verticesAfter:728, merged:0, message:"Vertices merged"` — zero welds on 3 exactly-coincident verts. Root cause in `MeshOpsHandler.cpp:1258-1296`: the handler calls `UGeometryScriptLibrary_MeshRepairFunctions::WeldMeshEdges` (FMergeCoincidentMeshEdges, boundary-EDGE weld) instead of a coincident-VERTEX weld, so on a closed mesh (no open boundary edges) it is a silent no-op while reporting success. Documented as "Merge nearby vertices" with a "Merge distance tolerance" param — the implementation contradicts that contract. Also noted: `remove_degenerates` returns no removed-count (secondary ergonomic gap, sibling of E-geometry-deformer-echo-mesh-counts).
