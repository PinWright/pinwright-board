---
id: E-append-buffers-cannot-express-coincident-triangles
title: "`append_buffers` cannot express exactly-coincident triangles — the engine refuses them as non-manifold at append — so the COPLANAR branch of `health.selfIntersections` has no live repro path from the client"
status: OPEN
severity: Low
category: ergonomic
tags: [pwmodel, append_buffers, health, selfIntersections, coplanar, membrane, testability, no-repro-path, verification-gap, engine-limit]
encounters: 1
lastSeen: 2026-08-28
---

# A shipped branch of a correctness check cannot be exercised from the client, and the obstacle is upstream of the plugin

**This is a testability gap, not a known bug.** Nothing here reports a wrong answer, and no author
is blocked. `health.selfIntersections` is believed correct on both its branches. The defect is that
only **one** of the two can be reached through the RPC surface, so the other is shipped code that
the client cannot prove, disprove, or regression-test.

`GeometryUtils::MeasureMeshSelfIntersection` (added by
`B-pwmodel-health-blind-to-interior-membrane` `#2` / `B-pwmodel-health-no-self-intersection` `#2`)
answers one question — *is this surface an embedded boundary?* — through two cases:

| branch | geometry | reachable from the client? |
|---|---|---|
| **transversal** | triangles that CROSS | **yes** — verified live, twice |
| **coplanar** | triangles that COINCIDE | **no** — see below |

The coplanar branch is the one the membrane ticket was actually filed about: *"a membrane across a
bore is two oppositely wound fans whose volumes cancel exactly"*, and it is the reason
`SetReportCoplanarIntersection(true)` was forced on at all, since the engine's default tri-tri
routine returns false for coplanar pairs. It is the branch with no way to test it.

## The obstacle is the engine, not the plugin — a fixer needs to know this before opening `GeometryUtils.cpp`

Two coincident, oppositely wound fans require two triangles occupying the same three vertices.
`append_buffers` is the only op in the format that takes explicit vertex and triangle buffers, and
the **engine's `UDynamicMesh` refuses those triangles when the buffers are appended**, before any
health measurement runs. Measured 2026-08-28 on `b79ba53e`, a 12-vertex prism with a mid-ring
membrane written as two coincident quads `(4,5,6),(4,6,7)` plus their reverses `(4,6,5),(4,7,6)`:

```
[MODEL_COMPILE_FAILED] [PWMODEL_OP_FAILED] line 3, col 5: 'append_buffers' failed
[MESH_APPEND_FAILED]: append_buffers: the engine refused 4 of 24 triangle(s) (it would have
accepted 20). The mesh has been restored to its state before the append, so nothing was
partially applied. The engine reported: AppendBuffersToMesh: Triangle cannot be added because
it would create invalid Non-Manifold Mesh Topology (x4)
```

Exactly the 4 membrane triangles were refused and the other 20 accepted, so this is not a malformed
document — it is `FDynamicMesh3` declining duplicate triangles by design. PinWright's own handling
of the refusal is **good** and is not what this ticket asks to change: it names the count, names
what would have been accepted, states that nothing was partially applied, and quotes the engine.

The near-miss variants do not help either, and the reason is worth recording so nobody re-derives
it:

- **Membrane with its own duplicate vertices** (same positions, different indices) — coincident, but
  it becomes its own connected component, and the measurement is deliberately **per shell**, so it
  is correctly skipped rather than counted.
- **Membrane split on the other diagonal** (`(4,5,6),(4,6,7)` against `(4,7,5),(5,7,6)`) — accepted
  by the engine and coplanar-overlapping, but every pair shares an edge with its partner, and
  `bIgnoreTopoConnected=true` ignores topologically connected pairs. Reports 0, correctly.

So the coplanar branch is reachable only by geometry that is simultaneously coincident,
single-shell, and topologically disconnected from its partner — which `append_buffers` cannot build
because the engine rejects the first property.

## What IS verified, so the gap is scoped honestly

The transversal branch is solid and independently confirmed twice, on different fixtures:

- A crossed ("bowtie") square prism against an **identical-topology** uncrossed control — same
  12-triangle index list, same 8 vertex positions reordered — reports `selfIntersections: 4` in 1
  shell with witness `(-0, 0, 100)` against the control's `0`, the analytic crossing point being
  `(0, 0, 100)`. Every other health field reads green on the crossed mesh.
- `B-pwmodel-health-no-self-intersection` `#3` reports an axis-crossing revolve at **3341** pairs
  against a clean revolve's 0.

Neither touches the coplanar path. The only coverage it has is the C++ fixture
`Tests/Model/TestPwModelSelfIntersection.cpp`, which builds the membrane in-process where the
engine's append-time manifold check does not apply.

## What it should do

Any one of these closes it; they are listed cheapest first.

1. **Say so in the docs.** `docs/pwmodel-format.md`'s `PWMODEL_SELF_INTERSECTING_SURFACE` row and
   the Mesh health section should record that a coincident membrane cannot be authored through
   `append_buffers`, so a reader who tries to reproduce the headline case does not conclude the
   check is broken. This costs nothing and removes the worst outcome, which is someone "fixing" a
   working measurement.
2. **Give `append_buffers` a way to bypass the manifold check.** The engine's own
   `FDynamicMesh3::AppendTriangle` returns an `EMeshResult` that a non-manifold-tolerant append
   could act on, and the op already reports partial refusals precisely. An opt-in
   (`allow_non_manifold=true`, refused by default) would make the membrane authorable and would also
   serve any document that legitimately wants a doubled surface.
3. **Expose the measurement on a mesh the client can build another way** — e.g. wire
   `geometry.check_health` to `MeasureMeshSelfIntersection` (deliberately out of scope in
   `B-pwmodel-health-no-self-intersection` `#2`, and tracked as
   `E-geometry-check-health-blind-to-self-intersection`) and reach the coplanar case through a
   `geometry.*` construction instead.

## Distinct from related tickets

- `B-pwmodel-health-blind-to-interior-membrane` (DONE) is the ticket whose fix this cannot fully
  verify. Its `#3` closure records the same obstacle in one paragraph and closes on the transversal
  evidence, which was the right call — the generalisation it asked for is proven. This ticket exists
  so the residual is tracked somewhere a fixer will find it, rather than only inside a closed
  ticket's closing note.
- `B-pwmodel-health-no-self-intersection` (DONE) shares the measurement and is likewise closed on
  transversal evidence only.
- `E-geometry-check-health-blind-to-self-intersection` is about a **different verb** not carrying
  the signal at all. Option 3 above would incidentally serve both; they are not the same ask.
- `B-merge-vertices-welds-edges-not-vertices` (IN-REVIEW) touches welding and non-manifold
  vocabulary but is a semantic mismatch in `geometry.merge_vertices`. Word collision only.

severity rationale: impact=no author is blocked and no response is wrong today; the cost is that one branch of a correctness signal can regress silently, and the failure it would let through is precisely the false green (`isClosed && signedVolume > 0` on a solid with a membrane) that two tickets were opened to close — partially offset by C++ fixture coverage and by the transversal branch being well verified x reach=one branch of one measurement, encountered only by someone verifying or modifying `MeasureMeshSelfIntersection`, not on any authoring path -> Low. Not cosmetic: the whole class of defect this project keeps paying for is a green metric over broken output, and this is a metric whose greenness is unfalsifiable from outside.

## History
- `#1-found-while-verifying-the-membrane-fix` `OPEN` reporter — 2026-08-28, UE 5.8, PinWright at
  `b79ba53e` in this checkout. Found while verifying `B-pwmodel-health-blind-to-interior-membrane`
  behaviourally. Attempting the ticket's literal headline geometry — two exactly coincident,
  oppositely wound fans inside one shell, as a 12-vertex prism with a doubled mid-ring membrane —
  fails at `append_buffers` with `[MESH_APPEND_FAILED]` / `AppendBuffersToMesh: Triangle cannot be
  added because it would create invalid Non-Manifold Mesh Topology (x4)`, refusing exactly the 4
  membrane triangles and accepting the other 20. Obstacle is `FDynamicMesh3` declining duplicate
  triangles, upstream of the plugin; PinWright's refusal message is precise and is not at fault.
  Two near-miss constructions were tried and both correctly report 0 for design reasons rather than
  reaching the branch: duplicate-vertex membranes become their own component and the measure is
  per-shell; opposite-diagonal membranes are topologically connected and `bIgnoreTopoConnected=true`
  ignores them. The transversal branch was verified in the same session with an identical-topology
  control (crossed prism 4 pairs, witness `(-0, 0, 100)` against analytic `(0, 0, 100)`; uncrossed
  control 0) and independently at 3341 pairs in `B-pwmodel-health-no-self-intersection` `#3`, so the
  gap is confined to the coplanar case. Classified ergonomic/testability, not a bug: no wrong answer
  was observed on any surface. Deduped before filing: `append_buffers` appears in 9 tickets and
  `coplanar` in 3, none covering this; no ticket owns the append-time manifold refusal.
