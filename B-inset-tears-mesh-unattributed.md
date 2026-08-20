---
id: B-inset-tears-mesh-unattributed
title: "Three shipped docs still say a .pwmodel inset tears a mesh open with the compile reporting nothing — the merged-mesh health stage now reports it, but no diagnostic attributes the tear to the op that caused it"
status: OPEN
severity: Low
category: bug
tags: [pwmodel, inset, boundary-edges, docs-stale, diagnostics, attribution, PWMODEL_MESH_NOT_CLOSED, watchtower]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# The silence is fixed; the docs saying it is silent are not, and the op still says nothing itself

## The silence half is closed

`FCompiler::ValidateMergedMesh` measures merged-mesh health at
`Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:2408-2412` and raises
`PWMODEL_MESH_NOT_CLOSED` (Warning) at `:2414-2424`. It is stage 4, called unconditionally at
`:2680` — so on `model.compile` **and** `model.validate` — and the counts reach the response as
`health.isClosed` / `health.boundaryEdges` / `degenerateTriangles` / `nonManifoldVertices` /
`componentCount` at
`Source/PinWrightGeometry/Private/Handlers/Model/ModelCompileHandler.cpp:468-477`. The stage's own
comment at `PwModelCompiler.cpp:2386-2391` names this exact case as what it closed: "at one point,
272 boundary edges: an open shell shipped as a successful asset with `model.compile` answering
`success:true` and saying nothing." That landed in `909c6d76` (2026-08-20 13:02).

## Three shipped documents still say otherwise

- `Examples/pwmodel/watchtower.pwmodel:176-185` — "`inset` TEARS THE MESH OPEN. On this parapet it
  leaves 272 boundary edges on its own … **Nothing in the compile says so: it reports success, and
  only `geometry.check_health` sees the hole.**"
- `Docs/wiki-src/model.examples.op-coverage.md:83` — "The compile reports success; only
  `geometry.check_health` sees the hole." Written 15:55 on 2026-08-20, ~3h **after** the health stage
  landed.
- `Docs/wiki-src/model.examples.md:95` — "`watchtower` records 272 boundary edges from `inset` alone
  on its parapet, **reported as a clean compile**."

All three now understate what the tool reports, which sends an author to a second verb they no longer
need.

## What is still genuinely missing

1. **No per-op attribution.** `GeometryOps::Inset`
   (`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryOps_Modeling.cpp:1021-1066`) performs
   no closedness check and — unlike `extrude` (`:1011`) and `offset_faces` (`:1147`) — never calls
   `GeometryOpsModeling_WarnOpenMeshPartialSelection`. It is the only face op without it. An author
   reading a merged-mesh warning cannot tell which op tore the surface.
2. **The format reference is silent.** `Docs/pwmodel-format.md` mentions `inset` only at `:501`,
   `:817`, `:831` (face-direction matching, polygroup options, why `outset` has no `reproject`), and
   the op description in the table is just "Shrink faces inward in their own plane"
   (`PwModelParser.cpp:538`).
3. The compile still returns `success: true` — by design, since the code is a Warning
   (`PwModelCompiler.cpp:2393-2401`). Noted so the next reader does not file it again.

## Not the inset/outset direction ticket

`B-geometry-inset-outset-direction-swapped` (IN-REVIEW, High) is the `geometry.inset` /
`geometry.outset` **RPC handlers** passing an inverted `Options.Distance` in `MeshOpsHandler.cpp`.
That sign was corrected on 2026-07-01 (`2774852d`), and the model path carries the corrected
convention at `GeometryOps_Modeling.cpp:1030-1034` ("Engine convention: POSITIVE Distance insets
inward" … `Options.Distance = Params.Distance;`). Different surface, different symptom, and the
direction bug predates and cannot explain the 272 edges.

**Fix:** correct the three stale sentences to say the compile now reports `health.boundaryEdges` and
raises `PWMODEL_MESH_NOT_CLOSED`; add the `GeometryOpsModeling_WarnOpenMeshPartialSelection` call to
`Inset` so the tear is attributed to the op; and give `inset` a line in `pwmodel-format.md` saying it
can open a closed solid.

## History
- `#1-silence-closed-docs-and-attribution-not` `OPEN` reporter — Three shipped documents state that a `.pwmodel` `inset` opens a closed solid with no compile-side signal: `Examples/pwmodel/watchtower.pwmodel:176-185` ("272 boundary edges on its own … Nothing in the compile says so"), `Docs/wiki-src/model.examples.op-coverage.md:83`, and `Docs/wiki-src/model.examples.md:95` ("reported as a clean compile"). The silence half is no longer true: `FCompiler::ValidateMergedMesh` measures health at `PwModelCompiler.cpp:2408-2412` and raises `PWMODEL_MESH_NOT_CLOSED` at `:2414-2424` on every compile and validate (`:2680`), with `ModelCompileHandler.cpp:468-477` publishing `health.isClosed`/`health.boundaryEdges`; the stage comment at `:2386-2391` cites this exact 272-edge case as what it closed, landed `909c6d76`. `op-coverage.md:83` was written ~3h after that stage landed and still carries the old sentence. What survives: `GeometryOps::Inset` (`GeometryOps_Modeling.cpp:1021-1066`) performs no closedness check and is the only face op that never calls `GeometryOpsModeling_WarnOpenMeshPartialSelection` (which `extrude` `:1011` and `offset_faces` `:1147` do), so a merged-mesh warning cannot be attributed to the op that caused it; and `Docs/pwmodel-format.md` never mentions the behaviour under `inset` (only `:501`, `:817`, `:831`, with the op description at `PwModelParser.cpp:538` reading "Shrink faces inward in their own plane"). The compile still returns `success:true` by design, the code being a Warning (`:2393-2401`). Distinct from `B-geometry-inset-outset-direction-swapped` (IN-REVIEW): that is the RPC handlers' inverted `Options.Distance` in `MeshOpsHandler.cpp`, corrected 2026-07-01 in `2774852d`, with the model path carrying the correct convention at `GeometryOps_Modeling.cpp:1030-1034` — it predates and cannot explain the 272 edges. No compile was run to re-measure the 272 figure; only the code path and the documentation were verified.
