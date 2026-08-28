---
id: B-pwmodel-health-blind-to-interior-membrane
title: "The published health gate cannot see a surface spanning a solid's interior: two oppositely wound fans cancel in signedVolume and every other field reads clean"
status: DONE
severity: High
category: bug
tags: [pwmodel, health, isClosed, signedVolume, false-green, membrane, revolve, model.validate]
encounters: 1
lastSeen: 2026-08-28
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

- `#3-verified-behaviourally-on-the-built-binary` `DONE` verifier — 2026-08-28, against the
  `b79ba53e` build, `model.validate` only, synthetic `append_buffers` documents, no host content.
  **The signal exists, is measured, and fires on the exact false green this ticket describes.**
  Probe: a crossed ("bowtie") square prism — 8 vertices, 12 triangles, the two diagonal walls
  passing through each other at the axis. Control: the SAME 12-triangle index list with the same
  8 positions reordered into a plain prism, so topology, triangle count and bounds are identical
  and only the crossing differs.

  | document | tris | isClosed | boundaryEdges | orientCons. | degenTris | nonManifold | signedVolume | **selfIntersections** |
  |---|---|---|---|---|---|---|---|---|
  | crossed prism | 12 | true | 0 | true | 0 | 0 | 0 | **4** (1 shell) |
  | same topology, uncrossed control | 12 | true | 0 | true | 0 | 0 | -8,000,000 | **0** |
  | crossed prism + a disjoint 200-cube part | 24 | true | 0 | true | 0 | 0 | **+8,000,000** | **4** (1 shell) |
  | nested hollow shell (200-box + 100-box) | 24 | true | 0 | true | 0 | 0 | 9,000,000 | **0** |

  Row 3 is the ticket's case verbatim: **every** field of the old documented gate reads green —
  `isClosed: true` and `signedVolume: 8,000,000 > 0`, with `boundaryEdges`, `degenerateTriangles`,
  `nonManifoldVertices` and `inconsistentEdges` all zero and `orientationConsistent: true` — while
  `selfIntersections: 4` / `selfIntersectingComponents: 1` report the interior surface. Row 4 is the
  declined-by-design control: two nested shells are separate components, so the per-shell measure
  correctly stays 0 and `PWMODEL_UNUNIONED_OVERLAP` covers it instead.
  `PWMODEL_SELF_INTERSECTING_SURFACE` fires as a warning with the witness coordinate **(-0, 0, 100)**
  — the analytic crossing point of the two diagonal walls is exactly (0, 0, 100) — and names the
  three usual causes plus why appended siblings are a different code.

  Not a warning-in-place-of-a-fix: the measurement is new state on `health`, not narration of
  unchanged output, and it separates a self-intersecting mesh from a topologically identical clean
  one that no pre-existing field distinguishes.

  **One input path could not be exercised, recorded so nobody re-reads this as fully covered:** the
  ticket's literal geometry — two *exactly coincident* oppositely wound fans — cannot be authored
  through `append_buffers` at all. The engine refuses the duplicate triangles up front:
  `[MESH_APPEND_FAILED] ... the engine refused 4 of 24 triangle(s) ... Triangle cannot be added
  because it would create invalid Non-Manifold Mesh Topology (x4)`. So the coplanar-coincident
  branch (`SetReportCoplanarIntersection(true)`) is verified only by the crossing case above and by
  the shipped tests, not by a live coincident-fan document. Closing on the crossing evidence, which
  is the generalisation the ticket asked for ("any future op that produces an interior surface").
