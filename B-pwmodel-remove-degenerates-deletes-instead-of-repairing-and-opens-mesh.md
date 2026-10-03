---
id: B-pwmodel-remove-degenerates-deletes-instead-of-repairing-and-opens-mesh
title: "`remove_degenerates` never repairs: repair_or_delete DELETES and leaves 11 boundary edges, repair_or_skip changes nothing, and its default min_triangle_area=0.001 is 1000x the degenerate epsilon and shredded 5,284 live triangles"
status: DONE
severity: Medium
category: bug
tags: [pwmodel, model-compile, model-validate, remove_degenerates, degenerate-triangles, mesh-not-closed, defaults, weapons]
encounters: 1
costly: 1
lastSeen: 2026-09-06T06:45:00Z
---

# The only op named for the problem cannot solve it, and at its own defaults it destroys the part

Two separate faults, both measured on one part (`SM_WPN_AR.pwmodel`'s `part rail`: a 37 x 4.2 x 1.0
bar, `bevel distance=0.12`, 34 cross-slots cut through it, 13,106 triangles carrying 5 zero-area
ones).

## 1. `mode` never repairs — it deletes, and the part comes back open

| op | tris | degenerate | isClosed | boundaryEdges |
|---|---|---|---|---|
| (none) | 13,106 | 5 | true | 0 |
| `remove_degenerates mode=repair_or_delete min_triangle_area=0.000001 min_edge_length=0.000001` | 13,101 | 0 | **false** | **11** |
| `remove_degenerates mode=repair_or_skip` (same thresholds) | 13,106 | **5** | true | 0 |
| `remove_degenerates mode=delete_only` + `fill_holes` | 13,106 | **5** | true | 0 |
| `weld_vertices tolerance=0.0001` | 13,106 | 5 | true | 0 |
| `weld_vertices tolerance=0.001` | 13,106 | 5 | true | 0 |

`repair_or_delete` removed exactly the 5 and opened 11 edges, so the "repair" half of the mode did
not fire on a single one of them. `repair_or_skip`, documented as "collapses it if it can", changed
nothing at all — the two rows together say the collapse path never runs on this input. `fill_holes`
closes the holes by putting the same 5 slivers back (13,101 -> 13,106, degenerate 0 -> 5), which is
the round trip that proves these triangles are load-bearing topology and not stray geometry.

Net: **a document that produces a zero-area triangle has no in-format way to remove it and stay
closed.** `health.isClosed && signedVolume > 0 && selfIntersections === 0` is the published gate, so
"clean the degenerates" and "pass the gate" are mutually exclusive.

## 2. The default thresholds are 1000x the epsilon the compiler itself uses

`model.describe_ops {op: "remove_degenerates"}` gives `min_triangle_area` default **0.001** and
`min_edge_length` default **0.0001**, both absolute world units. `PWMODEL_DEGENERATE_GEOMETRY`
defines a degenerate as **area < 1e-06**. On this model (1 uu = 9.23 mm) the op at its own defaults
deleted **5,284 triangles from the rail and 36 from the barrel** — real, non-degenerate geometry —
and the whole model came back `isClosed: false`, 3,418 boundary edges, 255 components (from 23),
181 bowtie vertices, while reporting `degenerateTriangles: 0`. Every one of those numbers is a
"success". A caller who trusts the default and gates only on the degenerate count ships a shredded
mesh.

The parameter help says the thresholds are absolute and that on a metre-authored mesh they are
"effectively zero" — the opposite failure to this one, which is a centimetre-authored mesh where
they are enormous. `model.examples.op-coverage` already calls the op a "measured no-op where it
looked right" and says the fix belongs at source; that note does not mention that the defaults can
delete thousands of live triangles.

## What was called

`model.validate {filePath: "Content/FPS/Weapons/Meshes/SM_WPN_AR.pwmodel"}` and six one-line
variants of the isolated `part rail`, UE 5.8.

## What was expected

- `repair_or_delete` to collapse a near-collinear sliver onto its neighbours (the classic edge
  collapse) and keep the mesh closed, which is what "repair" means and what the mode names promise;
  at minimum, for `repair_or_skip` to differ from a no-op.
- Defaults scaled to the same epsilon the compiler's own diagnostic uses, or a diagnostic when the
  op deletes more than a token number of triangles, or `PWMODEL_MESH_NOT_CLOSED` promoted to an
  error when an op that claims to clean geometry is what opened it.

## Workaround

Do not use the op. Both degenerate sources on this asset were fixed at source — see
`B-pwmodel-compound-boolean-tool-oversubdivides-and-leaves-slivers` (one boolean per cut instead of
a 34-box compound tool) and the gas-block case in the same file (a `union { }` block instead of a
part-level `bevel` with a filter box that necessarily contained buried edges).

## Root cause — guess, no source read taken

`repair_or_delete` looks like a straight call into the engine's degenerate-triangle removal with the
collapse path either not wired or bailing whenever a collapse would change the boundary of a
polygroup; the identical triangle counts for `repair_or_skip` and no-op suggest the skip mode never
attempts anything. **Inference from the response numbers; no plugin source was opened.**

## Severity

**Medium.** The wrong default is loud once you look (the mesh opens), and the source-side fix exists
for the cases seen so far — but it is a verb whose entire purpose is unavailable, and its default
invocation is destructive on centimetre-scale content, which is what this whole project is.

## Related

- `B-pwmodel-compound-boolean-tool-oversubdivides-and-leaves-slivers` — where the slivers came from.
- `B-blossom-pole-ring-degenerate-triangles` — another source of degenerates in the format.
- `model.examples.op-coverage` (`remove_degenerates` row) — documents the no-op behaviour and the
  cone-apex case where deleting is the wrong fix; it should also carry the default-threshold trap.

## History
- `#1-filed` `OPEN` reporter — Filed while closing WEAPONS review 04 defect 3. Six variants measured
  on the isolated rail plus one whole-model run at the op's defaults (23,266 -> 17,946 triangles,
  isClosed false, 3,418 boundary edges). All numbers above are from `model.validate` on UE 5.8 in
  this checkout.
- `#2-repair-in-place` `IN-REVIEW` developer — Root cause confirmed in source: `GeometryOps::RemoveDegenerates`
  passed straight to the engine's `RepairMeshDegenerateGeometry`, whose only "repair" is a QEM
  `SimplifyToEdgeLength(min_edge_length)` with every boundary and seam constrained
  (`MeshRepairFunctions.cpp`, with the engine's own TODO "this would not resolve sliver triangles
  that have all long-edges"). A zero-area cap therefore never got repaired: `repair_or_delete`
  deleted it (opening the mesh), `repair_or_skip` left it. Fix
  (`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryOps_Modeling.cpp`, namespace
  `GeometryOpsRemoveDegenerates`): for every mode except `delete_only`, a plugin-side pass runs
  before the engine pass - a needle/point triangle (an edge <= max(min_edge_length,
  sqrt(min_triangle_area))) is edge-collapsed (orientation guard on the one-ring), a cap is
  repaired by splitting its long edge at the apex projection and collapsing the new vertex onto
  the apex (engine `SplitEdge`/`CollapseEdge`, so UV/normal seams and material IDs follow the
  engine's overlay bookkeeping); passes repeat until no progress (max 16). `repair_or_delete` now
  warns `remove_degenerates opened N boundary edge(s)` when its fallback deletion opens the mesh
  (not for `delete_only`, which the .pwmodel boolean cleanup uses deliberately). Default
  `min_triangle_area` 0.001 -> 0.000001 (= `GeometryUtils::DegenerateAreaEpsilon`) in
  `GeometryOps_Modeling.h`, `PwModelParser.cpp` (op vocabulary) and `MeshOpsHandler.cpp`
  (`geometry.remove_degenerates` docs). Docs: `docs/wiki-src/model.examples.op-coverage.md`,
  `docs/wiki-src/model.authoring.md`, comments in `Examples/pwmodel/watchtower.pwmodel` and
  `driftwood.pwmodel`, README row, CHANGELOG (behaviour change). Tests:
  `PinWright.Geometry.Ops.RemoveDegeneratesRepair.CapIsRepairedAndTheMeshStaysClosed`,
  `PinWright.Geometry.Ops.RemoveDegeneratesRepair.DefaultLeavesSmallRealTrianglesAlone`,
  `PinWright.Model.RemoveDegeneratesRepair.PointedConeApexIsWeldedClosed` (new file
  `Tests/Geometry/TestGeometryRemoveDegeneratesRepairs.cpp`); updated default pins in
  `PinWright.Geometry.Ops.ModelingOptions.DefaultsMatchTheEngineStructDefaults` (pin removed) and `PinWright.Geometry.Ops.ModelingOptions.DocumentedDivergencesFromTheEngineDefaults` (pin added) and
  `PinWright.Model.ModelingOps.NormalsAndRepairOpsPublishTheirEngineOptions`. Not re-measured on
  the original weapon rail (host asset); the synthetic cap fixture reproduces the reported shape. After
  review: the cap split-and-collapse also refuses when either replacement triangle would face
  opposite to the far triangle (a cap folded over its neighbour passed the unsigned area test);
  covered by the `[folded cap]` block of `CapIsRepairedAndTheMeshStaysClosed`. The opened-edge
  warning now counts boundary edges after the plugin repair, and `delete_only` reads the mesh
  through `ProcessMesh` instead of an empty `EditMesh`.
- `#3-verified-linux` `DONE` tester — Verified on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d). run3/full, non-skipped: `PinWright.Geometry.Ops.RemoveDegeneratesRepair.CapIsRepairedAndTheMeshStaysClosed` covers the cap repair across modes. repair_or_delete and repair_or_skip repair the cap and keep the mesh closed with no opened-edge warning, while before the fix one deleted it and the other was a no-op. A folded cap is refused, and the repair_or_delete fallback warns `opened N boundary edge(s)`. `.DefaultLeavesSmallRealTrianglesAlone` covers the default min_triangle_area of 1e-6, equal to the degenerate epsilon. `PinWright.Model.RemoveDegeneratesRepair.PointedConeApexIsWeldedClosed` covers a cone that had 64 boundary edges before the fix and is now closed. The updated default pins also passed (`ModelingOptions.DefaultsMatchTheEngineStructDefaults`, `.DocumentedDivergencesFromTheEngineDefaults`, `Model.ModelingOps.NormalsAndRepairOpsPublishTheirEngineOptions`). Every expected item is met: repair keeps the mesh closed, repair_or_skip differs from a no-op, the defaults sit at the compiler's epsilon, and an opened mesh is diagnosed. Coverage limit: a synthetic cap fixture; the SM_WPN_AR rail was not re-measured.
