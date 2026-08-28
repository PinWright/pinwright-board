---
id: E-pwmodel-orphan-vertices-not-reported
title: "meshVertexCount counts vertices the reported bounds now excludes, and nothing names an orphan vertex"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [pwmodel, health, orphan-vertex, meshVertexCount, bounds, FMeshHealth, missing-signal]
encounters: 1
lastSeen: 2026-08-27
---

# Two fields describing the same mesh can now disagree, by design

`B-pwmodel-bounds-orphan-vertex-after-boolean` is fixed: the reported `bounds` is reduced over
triangles, so a vertex a boolean left referenced by nothing no longer widens the box.

`meshVertexCount` still counts that vertex. The two fields therefore describe the same mesh and
disagree about what is in it -- correctly, but with nothing saying so.

An orphan vertex is itself a mesh-health signal worth having: it means an op removed geometry without
cleaning up, and it is the tell for the class of boolean leftovers that produced the original ticket.

**Fix:** an orphan/unreferenced-vertex count on the `health` block. That is a new `FMeshHealth` field
(`GeometryUtils.h/.cpp`), a new `FPwModelCompileResult` field, and handler emission -- four shared
files, which is why the bounds fix did not include it.

## History
- `#1-left-by-the-bounds-fix` `OPEN` reporter -- Recorded by the agent that fixed
  `B-pwmodel-bounds-orphan-vertex-after-boolean`, which deliberately kept its change to the reporting
  path rather than widening into the health schema.
- `#2-orphan-count-on-the-health-block` `IN-REVIEW` developer -- "Added `FMeshHealth::UnreferencedVertices`
  in GeometryUtils.h/.cpp counting live vertices `FDynamicMesh3::IsReferencedVertex` rejects, folded into
  the existing bowtie sweep so the walk stays one pass; threaded it as `FPwModelCompileResult::
  MeshUnreferencedVertices` (-1 sentinel) through PwModelCompiler.h/.cpp's ValidateMergedMesh and emitted
  it as `health.unreferencedVertices` in ModelCompileHandler.cpp. Reported, not judged: it is in no
  verdict, because the triangle-driven bake means an orphan reaches no asset. Regression coverage in
  Tests/Model/TestPwModelReferencedBounds.cpp, test id
  `PinWright.Model.Bounds.TheOrphanTheBoxExcludesIsCountedOnTheHealthBlock`:
  it asserts 0 on the plain sheet, 1 on the sheet carrying one unreferenced vertex, and
  that the count accounts for the whole `meshVertexCount` difference between them. Documented in
  Docs/pwmodel-format.md and Docs/wiki-src/model.md. NOT compiled or run -- build/test is the
  verification pass. Follow-up worth a ticket: `geometry.check_health` measures the field but does not
  emit it, so the two callers of the shared walk now report different field sets."
