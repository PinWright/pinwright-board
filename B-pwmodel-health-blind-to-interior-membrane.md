---
id: B-pwmodel-health-blind-to-interior-membrane
title: "The published health gate cannot see a surface spanning a solid's interior: two oppositely wound fans cancel in signedVolume and every other field reads clean"
status: IN-REVIEW
severity: High
category: bug
tags: [pwmodel, health, isClosed, signedVolume, false-green, membrane, revolve, model.validate]
encounters: 1
lastSeen: 2026-08-27
---

# `isClosed && signedVolume > 0` returns green on a solid with a membrane through it

This is the general defect behind `B-revolve-closed-profile-fills-bore`. That ticket's geometry half is
fixed -- a closed profile no longer gets capped to the axis -- but the reason it went unnoticed is
untouched: **no published health field can see an interior membrane.**

The two axis fans are oppositely wound, so their contributions to `signedVolume` cancel exactly.
`isClosed` is true (the surface really is closed). `boundaryEdges` is zero. `orientationConsistent` is
true. `degenerateTriangles` is zero. The documented gate `isClosed && signedVolume > 0` passes, and the
solid has a disc through its bore.

So any future op that produces an interior surface -- a boolean leaving a shared wall, a sweep that
self-intersects and welds, an authored profile that doubles back -- gets the same green.

**Fix direction:** a closure check that also asks whether any surface spans the interior. Candidates: a
ray cast from an interior point that should hit exactly one surface and hits three; or a per-component
volume check that would catch two nested shells summing to a plausible total. Related to
`B-pwmodel-health-no-self-intersection`, which asks for the neighbouring signal -- worth designing the
two together rather than bolting on two independent checks.

## History
- `#1-generalised-from-the-revolve-fix` `OPEN` reporter -- Raised by the agent that fixed
  `B-revolve-closed-profile-fills-bore`, which verified the health fields read green both before and
  after its fix and therefore wrote every new assertion geometrically rather than against the gate.
- `#2-self-intersection-health-signal` `IN-REVIEW` developer -- Added the missing signal as ONE
  measurement covering this ticket and `B-pwmodel-health-no-self-intersection`: a surface that is
  not an embedded boundary, counted as triangle pairs that cross or coincide INSIDE one
  edge-connected shell. New `GeometryUtils::FMeshSelfIntersection` /
  `MeasureMeshSelfIntersection` (`Handlers/Geometry/GeometryUtils.h/.cpp`) - one
  `FMeshConnectedComponents` walk, then one `FDynamicMeshAABBTree3` +
  `FindAllSelfIntersections` per component with `bIgnoreTopoConnected=true` and a custom
  intersection fn that turns `SetReportCoplanarIntersection(true)` ON, which is what makes the
  membrane's two coincident fans visible at all (the engine's default tri-tri routine returns
  false for coplanar pairs). Degenerate triangles are excluded via `FQueryOptions.TriangleFilterF`
  because the SAT test answers `true` for every pair involving a zero-area triangle. Declined
  above 200k triangles and capped at 4096 pairs, both reported rather than silently approximated.
  Per-shell and never cross-shell: appended parts/siblings interpenetrate deliberately in this
  format and already have `PWMODEL_UNUNIONED_OVERLAP{,_PARTS}`, so pooling them would fire on a
  large fraction of correct documents. Wired into `PwModelCompiler::ValidateMergedMesh` ->
  `FPwModelCompileResult::MeshSelfIntersections` / `MeshSelfIntersectingComponents` /
  `bMeshSelfIntersectionTruncated` (-1 sentinels), published by `ModelCompileHandler` as
  `health.selfIntersections` / `selfIntersectingComponents` / `selfIntersectionsTruncated`
  (absent, never zeroed, when declined), and warned as the new
  `PWMODEL_SELF_INTERSECTING_SURFACE` with a witness coordinate. Warning, not error - report, do
  not refuse. Docs: `docs/pwmodel-format.md` diagnostics-table row plus the Mesh health section,
  where the gate is now `isClosed && signedVolume > 0 && selfIntersections === 0`. Tests:
  `Tests/Model/TestPwModelSelfIntersection.cpp`, four cases - the membrane (asserting the whole
  false green first, then the new field), crossing walls inside one shell, four controls that
  must stay at zero (nested hollow shell, the correctly spelled ring, a lathe whose axis cap is
  coplanar with its own base annulus, an open sheet) and the two documents through
  `model.validate`.
