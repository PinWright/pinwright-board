---
id: B-extrude-inset-empty-selection-whole-mesh
title: "geometry.extrude/inset/outset/offset_faces give no way to target a face (always operate on the whole mesh) and report unconditional success — on a closed solid extrude silently duplicates the whole mesh and inset is a no-op"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, extrude, inset, outset, offset_faces, whole-mesh, no-face-selection, silent-noop, success-no-effect, dynamic-mesh]
---

# `geometry.extrude` / `geometry.inset` / `geometry.outset` / `geometry.offset_faces` cannot target a face — they always operate on the whole mesh and report success even when the result is a whole-mesh duplicate (extrude) or a no-op (inset)

`geometry.extrude` (`MeshOpsHandler.cpp:352-378`), `geometry.inset`
(`388-413`), `geometry.outset` (`418-448`) and `geometry.offset_faces`
(`489-517`) each construct a **default-empty** `FGeometryScriptMeshSelection
Selection;` and hand it to the Geometry-Script face op:

```cpp
FGeometryScriptMeshSelection Selection;                 // empty == whole mesh
UGeometryScriptLibrary_MeshModelingFunctions::ApplyMeshLinearExtrudeFaces(
    Target.Mesh, ExtrudeOptions, Selection, nullptr);   // extrude (line 368)
...
UGeometryScriptLibrary_MeshModelingFunctions::ApplyMeshInsetOutsetFaces(
    Target.Mesh, Options, Selection, nullptr);          // inset  (line 402)
```

In Geometry Script an empty `FGeometryScriptMeshSelection` means **"the entire
mesh"** (`ApplyMeshLinearExtrudeFaces` adds every triangle when
`Selection.GetNumSelected()==0`, MeshModelingFunctions.cpp:613-619;
`ApplyMeshInsetOutsetFaces` is identical, 815-821). Because **the RPC surface
exposes no way to pass a face selection**, a caller can never target a single
face — every call operates on the whole mesh. On a *closed* solid (e.g. a
`create_box`, which has no boundary edges) that is degenerate:

- **`extrude`** offsets the closed component as a full solid
  (`Extruder.bOffsetFullComponentsAsSolids = Options.bSolidsToShells`, default
  `true`, MeshModelingFunctions.cpp:603) with no boundary to stitch a side wall
  to, so it just **duplicates the whole mesh** offset along the direction. The
  result is two disconnected stacked shells, not a face raised on the existing
  solid. The signature is an **exact doubling** of vertex/triangle counts on
  every call (8→16→32 verts, 12→24→48 tris), which the box modeling task
  observed and misread as "the extrude added geometry."
- **`inset`** of the whole closed mesh has no boundary loop to inset, so it is a
  **silent no-op** — the count is completely unchanged (8v/12t before and after)
  — yet it returns `"Inset applied"`.

Both report unconditional `success` with no `changed` signal, so the caller
cannot tell that the documented operation ("Extrude faces", "Inset faces …
shrink inward") did not happen. This is the **missing-capability +
success-with-no-effect / wrong-effect** tool-bug class on a valid, documented
input.

## Root cause (corrected)

The earlier "just build a select-all-faces selection" framing was a
**misdiagnosis**: `CreateSelectAllMeshSelection(Triangles)` adds every triangle
(MeshSelectionFunctions.cpp:280-287), which the engine converts straight back to
the identical all-triangles array via `ConvertToMeshIndexArray`
(MeshModelingFunctions.cpp:622) — it is **functionally identical to the
empty-selection whole-mesh path** the engine already runs, so it would reproduce
the exact same degenerate result (extrude still duplicates, inset still no-ops).
Select-all is NOT the fix. The real defect is two-fold: (1) there is **no way to
target a face** on a closed solid, and (2) the handlers report success even on a
whole-mesh-duplicate or no-op.

This is a genuinely wrong *result*, not just a missing count echo — it is
distinct from `E-geometry-deformer-echo-mesh-counts` (which lists extrude/inset
only as deformers that should echo counts and assumes they work), and distinct
from `B-boolean-subtract-ignores-tool-offset` (a create_* double-applied
transform; different methods, different root cause).

## Repro (verbatim, replay-confirmed against mcp__editor-automation__call)

```
geometry.create_box {name:"ReplayPedestal", location:{x:0,y:0,z:0}, width:120, height:120, depth:200}
geometry.get_mesh_info {actorName:"ReplayPedestal"}
  -> {vertexCount:8, triangleCount:12}                         baseline (closed box)

geometry.inset {actorName:"ReplayPedestal", distance:20}
  -> {operation:"inset", distance:20, message:"Inset applied"} success
geometry.get_mesh_info {actorName:"ReplayPedestal"}
  -> {vertexCount:8, triangleCount:12}                         UNCHANGED — inset did nothing

geometry.extrude {actorName:"ReplayPedestal", distance:60, direction:{x:0,y:0,z:1}}
  -> {distance:60, message:"Extrude applied"}                  success
geometry.get_mesh_info {actorName:"ReplayPedestal"}
  -> {vertexCount:16, triangleCount:24}                        EXACT DOUBLE of baseline — duplicated shell, not a face extrude

geometry.extrude {actorName:"ReplayPedestal", distance:15, direction:{x:0,y:0,z:1}}
  -> {distance:15, message:"Extrude applied"}                  success
geometry.get_mesh_info {actorName:"ReplayPedestal"}
  -> {vertexCount:32, triangleCount:48}                        DOUBLED AGAIN
```

A correct "raise a recessed cap on a plinth" (inset top face by 20, then extrude
that inset face up by 60) would add a ring of vertices around the inset profile
and stitch a wall up to the cap — a non-power-of-two count well above 16. The
clean ×2 each call, plus the inset producing zero change, is the fingerprint of
the empty-selection-as-whole-mesh path on a closed solid.

## Fix

**1. Give the caller a way to target a face.** Add an optional `faceDirection`
`{x,y,z}` argument (alias `selectFaceNormal`) to `extrude`/`inset`/`outset`/
`offset_faces`. When present, build a real face selection with
`UGeometryScriptLibrary_MeshSelectionFunctions::SelectMeshElementsByNormalAngle`
(Normal = the supplied direction, `EGeometryScriptMeshSelectionType::Triangles`,
tolerance from an optional `faceAngleTolerance`, default ~45°) and pass **that**
selection to the engine op instead of the default-empty one — so a caller can
extrude/inset only the top face of a closed solid (the documented "raise a cap
on a plinth" use case). When `faceDirection` is absent, behavior is unchanged
(whole-mesh), preserving backward compatibility.

> NOT a fix: `CreateSelectAllMeshSelection(Triangles)`. Select-all is the same
> all-triangles array the empty-selection path already runs (see Root cause), so
> it changes nothing — extrude still duplicates the closed solid, inset still
> no-ops. The capability that is actually missing is a *subset* selection.

**2. Report whether anything changed.** In all four handlers capture
`vertexCount`/`triangleCount` before and after (the handler already holds
`Target.Mesh`; `UGeometryScriptLibrary_MeshQueryFunctions::GetVertexCount`/
`GetTriangleCount`), and put `vertexCount`, `triangleCount`, and a `changed`
flag (`true` iff either count moved) on the response, plus `facesSelected` (the
selected-triangle count, 0 when whole-mesh) so a no-op or whole-mesh-duplicate
is detectable instead of being silently reported as success. The post-op
count-echo portion overlaps `E-geometry-deformer-echo-mesh-counts` (which lists
these same deformers among those that should echo counts); satisfying it here
also satisfies that ergonomic ask for these four verbs.

**Workaround:** none clean from the RPC surface (pre-fix) — there is no way to
pass a face selection, and on a closed primitive every extrude silently
duplicates the mesh. A caller can only detect the misbehavior by diffing
`get_mesh_info` before/after and noticing the exact doubling / zero-change.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed against
  mcp__editor-automation__call on a `create_box` (120×120×200, 8v/12t closed
  solid). `geometry.inset {distance:20}` returns `"Inset applied"` but leaves the
  mesh at 8v/12t (silent no-op). `geometry.extrude {distance:60, dir 0,0,1}`
  returns `"Extrude applied"` and **exactly doubles** the mesh to 16v/24t; a
  second `extrude {distance:15}` doubles it again to 32v/48t. Root cause: both
  handlers pass a default-empty `FGeometryScriptMeshSelection` (= whole mesh) to
  `ApplyMeshLinearExtrudeFaces` / `ApplyMeshInsetOutsetFaces`
  (MeshOpsHandler.cpp:366-369, 400-403), which on a closed solid duplicates the
  mesh (extrude) or does nothing (inset) rather than extruding/insetting a face.
  Surfaced by a "block out a stone pedestal from a box" task whose attempt agent
  read the ×2 doublings as proof the inset+extrude "added geometry" and converted
  the two-stacked-box result to a StaticMesh, believing it had built a plinth +
  cap. Same empty-selection pattern in `outset` (line 435) and `offset_faces`
  (line 505) — needs the same audit.
- `#2-reword` `OPEN` developer — Struck the misdiagnosed primary fix. Three
  validity lenses (correctness, adversarial, board-historian) confirmed the
  defect is real and has no workaround, but the original Fix #1 ("build a
  select-all-faces selection via `CreateSelectAllMeshSelection(Triangles)`") is a
  **verified no-op**: that helper adds every triangle (MeshSelectionFunctions.cpp:
  280-287), which the engine converts straight back to the identical all-triangles
  array the empty-selection path already runs (MeshModelingFunctions.cpp:612-622)
  — extrude would still duplicate the closed solid, inset still no-op. Reframed
  title + root cause to the true scope: the RPC surface exposes **no way to target
  a face**, and the handlers report success on a whole-mesh-duplicate/no-op.
  Corrected Fix: (1) optional `faceDirection` {x,y,z} face-subset selection via
  `SelectMeshElementsByNormalAngle`; (2) `changed` + post-op count echo
  (count-echo overlaps E-geometry-deformer-echo-mesh-counts for these four verbs).
- `#3-fix` `IN-REVIEW` developer — Implemented the reworded fix in
  `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Handlers/Geometry/MeshOpsHandler.cpp`.
  Added two shared helpers in the anonymous namespace: `BuildFaceSelection`
  (reads optional `faceDirection`/`selectFaceNormal` {x,y,z} + `faceAngleTolerance`,
  default 45°, and builds a real triangle subset via
  `UGeometryScriptLibrary_MeshSelectionFunctions::SelectMeshElementsByNormalAngle`
  — empty/whole-mesh when absent, backward compatible) and `ReportMeshChange`
  (echoes post-op `vertexCount`/`triangleCount` via
  `MeshQueryFunctions::GetVertexCount` + `UDynamicMesh::GetTriangleCount`, plus
  `facesSelected` and a `changed` flag set iff either count moved). Wired all four
  face ops — `geometry.extrude`, `geometry.inset`, `geometry.outset`,
  `geometry.offset_faces` — to snapshot counts before, run the engine op with the
  built Selection, then report. Added `#include "GeometryScript/MeshSelectionFunctions.h"`.
  Documented the new optional params in each handler's `RPC_PARAMS`. Note: the
  `MeshQueryFunctions` library `GetTriangleCount` accessor is commented out in
  UE 5.7, so triangle count uses `UDynamicMesh::GetTriangleCount()`.
  Regression tests added to
  `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Tests/World/TestGeometryHandlers.cpp`:
  `geometry.extrude.ReportsChangeAndNoSilentDuplicate` (drives the PRODUCTION
  handler on a real `create_box`: whole-mesh extrude must report `changed=true`,
  `facesSelected==0`, and the exact ×2 doubled `triangleCount` 24; `faceDirection`
  {0,0,1} must select a non-empty face SUBSET `0 < facesSelected < 12` and NOT
  whole-mesh-duplicate to 24) and `geometry.inset.WholeMeshNoOpReported`
  (whole-mesh inset of the closed box reports `changed=false` and an unchanged
  triangleCount). Reverting either the face-selection wiring or the count/changed
  echo fails these (the fields vanish or `faceDirection` is ignored → ×2 / 0
  selected). Not compiled/tested here per workflow (a later phase runs the suite).
