---
id: F-geometry-loop-cut-proper
title: "Reimplement geometry.loop_cut as a real edge-loop insertion (removed: it was a destructive plane cut, not a loop cut)"
status: OPEN
severity: Medium
category: feature
tags: [geometry, loop-cut, edge-loop, dynamic-mesh, reimplement, rpc-audit]
---

# Reimplement `geometry.loop_cut` properly (insert an edge loop, do not slice the mesh)

`geometry.loop_cut` was **removed** in the batch-2 RPC audit recorded in
[`E-rpc-audit-43-record`](E-rpc-audit-43-record.md) because its implementation
had nothing to do with an edge-loop cut. The capability the name advertises
(insert a ring of edges around the mesh, adding topology without destroying
geometry) is legitimately wanted and has no replacement on the current surface.

## What the removed version did wrong

The handler ran a **destructive plane cut** through the dynamic mesh: it sliced
the mesh with a plane and kept/discarded geometry on one side, then reported
success as a "loop cut". That is a different operation with the opposite
contract:

- A **plane cut** removes geometry. The mesh that comes out is not the mesh that
  went in, and the caller loses whatever fell on the cut side.
- A **loop cut** adds geometry. It inserts a new edge ring around the existing
  topology, leaving the mesh's shape and volume identical and giving the caller
  a new ring of edges to select, extrude, or bevel against.

An agent asking for a loop cut to add a control edge (the normal reason to reach
for the verb) got its model sliced in half instead, with a `success:true`. That
is why it was removed outright rather than left in place.

## Proper implementation

Insert an edge loop, do not cut. UE already implements exactly this in Modeling
Mode's **Edge Loop Insert** tool, which sits on top of a GeometryProcessing
group-edge insertion operator over `FDynamicMesh3`. The handler should drive that
operator on the actor's `UDynamicMesh` rather than hand-rolling topology edits:

- Resolve the target loop by the edge/group boundary the caller names, or by a
  "ring index + normalized position along the ring" pair, so the inserted loop is
  addressable and repeatable.
- Support inserting N evenly-spaced loops in one call (the common case is a
  single loop at 0.5).
- Assert the invariant that distinguishes this verb from the removed one: the
  mesh's triangle/vertex count goes **up**, and its bounds are unchanged. If the
  operator would remove geometry, fail loud instead of succeeding.
- Echo `vertexCount`/`triangleCount` on the result, matching the house
  convention for geometry mutators (see
  [`E-geometry-deformer-echo-mesh-counts`](E-geometry-deformer-echo-mesh-counts.md)).

**Workaround (partial):** `geometry.subdivide` adds edges, but globally and
uniformly. It cannot place one controlled loop where the caller wants a control
edge, and it multiplies the whole mesh's triangle budget to get there. There is
no current verb that inserts a single edge ring.

**Fix:** New handler alongside the other mesh ops in the geometry mesh-ops
handler where the removed method lived. Regression test: create a box, run
`loop_cut`, assert the vertex/triangle counts rose, the bounding box is
unchanged, and the mesh is still closed (the removed plane-cut version would fail
all three, which is the differential proof that the reimplementation is a
different operation and not the old one under a new name).

## History
- `#1-reimpl-after-audit` `OPEN` reporter - Filed to reinstate the wanted capability removed by the batch-2 RPC audit ([`E-rpc-audit-43-record`](E-rpc-audit-43-record.md)). The removed `geometry.loop_cut` performed a destructive plane cut through the mesh (slice-and-discard) and reported it as a loop cut, so a caller asking for an added control edge got a bisected model back with `success:true`. Real edge-loop insertion (add a ring of edges, geometry preserved, counts up, bounds unchanged) is genuinely missing: `geometry.subdivide` is the only near-neighbor and it subdivides globally rather than placing one controlled loop. Proper impl: drive the GeometryProcessing group-edge insertion operator that backs Modeling Mode's Edge Loop Insert tool over the actor's `UDynamicMesh`, resolve the target ring by name/index + position, fail loud if the op would remove geometry, and echo post-op vertex/triangle counts per the house convention.
