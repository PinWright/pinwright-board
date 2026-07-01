---
id: E-geometry-boolean-omits-result-vertex-count
title: "geometry.boolean_subtract/union/intersection echo resultTriangles + changed but omit the result vertexCount — a story that asks to confirm a 'sensible vertex/triangle count' after a cut still costs a separate get_mesh_info for the verts the boolean response could carry"
status: OPEN
severity: Low
category: ergonomic
tags: [geometry, boolean_subtract, boolean_union, boolean_intersection, mesh-info, readback, round-trip, response-shape, vertex-count, partial-echo, consistency]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# Boolean ops are partial count-echoers: they report `resultTriangles`/`changed` but never `resultVertices`, so the canonical "confirm the cut produced a sensible vertex AND triangle count" check still forces a `get_mesh_info` for the missing verts

`geometry.boolean_subtract` (and its `boolean_union` / `boolean_intersection`
siblings, all sharing the success branch at `BooleanHandler.cpp:165-188`) is a
**partial** count-echoer — the same nuance `E-geometry-deformer-echo-mesh-counts`
`#4` documented for `geometry.poke`, but on the boolean surface that ticket does
not cover. On success the response carries:

```
{ targetActor, operation, success:true,
  targetTriangles, toolTriangles, resultTriangles, changed }
```

It echoes `resultTriangles` (`ResultMesh->GetTriangleCount()`,
`BooleanHandler.cpp:137,173`) and a `changed` flag (`ResultTriCount != TargetTriCount`,
`:184-185`) — so the *triangle* count and the did-it-actually-cut signal come
back inline. **But it never echoes `resultVertices`** — there is no
`ResultMesh->GetVertexCount()` call anywhere in the handler. So a caller whose
intent is the textbook post-boolean sanity check ("confirm the result is a
watertight-ish solid with a **sensible vertex/triangle count**, non-zero, more
than a plain cylinder would have") gets the triangle half of that assertion free
but must fire a separate `geometry.get_mesh_info` purely to read the vertex
count the boolean handler already had in hand (`ResultMesh` is live at
`:135-137`).

## Why this is friction (not just normal granularity)

This is the same partial-echo response-shape gap that `E-geometry-deformer-echo-mesh-counts`
`#4` raised for `poke` (echoes `triangleCount`/`originalTriangles`, omits
`vertexCount`, forcing a `get_mesh_info` for the verts the story's success
criterion named) — but the boolean ops are **explicitly out of that ticket's
scope**: it is scoped to the deformer family (`twist`/`taper`/`bend`/`bevel`/
`shell`/…) plus the two `array_*` verbs, never mentions boolean ops, and its
`#8-fix` wired `SetMeshCountFields` onto those deformers in `MeshOpsHandler.cpp`/
`GeometryTransformHandler.cpp` — it never touched `BooleanHandler.cpp`. So the
boolean surface is a distinct, uncovered instance of the identical gap, and it is
worse-positioned than the deformers were: boolean ops *change* the vertex count
materially (a subtract/union re-tessellates the seam), making the post-op vertex
count more load-bearing than on a deformer that often preserves it, yet the
boolean response is the one that omits it.

It is **distinct from** `F-geometry-boolean-batch-tools` (the N-create + N-subtract
*call-count* gap — no batch verb) and from `B-boolean-subtract-ignores-tool-offset`
(the offset-tool no-op correctness bug, fixed) and from
`B-geometry-convert-static-mesh-no-disk-write` (the bake-persistence bug the judge
filed for this very task): those are call-count / correctness / persistence; this
is the response-shape omission of one field on an otherwise-clean call.

## What it should do

Echo `resultVertices` (and ideally a `vertexCount` alias matching the key
`get_mesh_info` uses) next to the `resultTriangles`/`changed` it already returns,
read off the same live `ResultMesh` the handler still holds at
`BooleanHandler.cpp:135-137` — one `ResultMesh->GetVertexCount()`. This folds the
"sensible vertex/triangle count" confirmation into the boolean call and
eliminates the post-cut `get_mesh_info`, mirroring the `#8-fix` already applied to
the deformer/array family (and the `#4` poke `vertexCount` addition). The
narrowest possible change; the data is in hand.

A docs-only floor (note on `docs/wiki-src/geometry.md` that the boolean response
carries `resultTriangles`/`changed` but **not** the vertex count, so confirming
vertices requires a follow-up `geometry.get_mesh_info`) would at least make the
round-trip expected rather than discovered — but the behavior fix is preferred
since the value is one accessor away on the held mesh.

**Workaround:** call `geometry.get_mesh_info {actorName}` after the boolean to
read `vertexCount` (the `triangleCount` and the cut-happened signal are already
in the boolean response as `resultTriangles`/`changed`).

## Friction evidence (this task — geometry "StonePlanter" hollow-planter build, 21 calls incl. 9 wiki-nav, outcome tool_bug, friction:"none")

Struggle audit of the hollow-stone-planter build (two cylinders → boolean_subtract
→ bevel → auto_uv → generate_collision → convert_to_static_mesh; all RPCs
first-try clean, no retries/errors/`python.execute`; the judge filed
`B-geometry-convert-static-mesh-no-disk-write` `#2` for the bake-persistence bug).
Story **step 4** explicitly asked to "Run get_mesh_info on the resulting
PlanterOuter and confirm it's a watertight-ish solid with a **sensible
vertex/triangle count** (non-zero, and more than a plain cylinder would have)."
The call log shows the boolean→readback pair:

- `geometry.get_mesh_info {PlanterOuter}` → baseline `74v/144t` (bare cylinder)
- `geometry.boolean_subtract {target:PlanterOuter, tool:PlanterInner, keepTool:false}`
  → ok; response carried `resultTriangles:336` + `changed:true` (the self-report
  quotes exactly this: *"confirmed topology grew from 144->336 tris (changed:true)"*)
  but **no** vertex count.
- `geometry.get_mesh_info {PlanterOuter}` → post-bool `170v/336t` — fired solely to
  read the `170` **verts** that step 4's "vertex/triangle count" assertion required
  and the boolean response omitted. The `336` tris in this readback was already in
  hand from the boolean's `resultTriangles`.

So the triangle half of the step-4 success criterion came free from the boolean
echo; the vertex half forced the extra `get_mesh_info`. Pure PROCESS overhead on a
clean call — a `resultVertices` echo would have folded the whole "sensible
vertex/triangle count" confirmation into the `boolean_subtract` response. The cost
recurs once per boolean cut whenever the verify step names the vertex count
(every "is this still a sane solid?" check after a subtract/union).

## Dedup

No existing ticket covers the boolean response's missing vertex echo:
`E-geometry-deformer-echo-mesh-counts` is scoped to deformers + `array_*` and its
fix never touched `BooleanHandler.cpp`; `F-geometry-boolean-batch-tools` is the
batch/call-count gap; `B-boolean-subtract-ignores-tool-offset` is the offset no-op
correctness bug; `B-geometry-convert-static-mesh-no-disk-write` (this task's
judge filing) is bake-persistence; `E-geometry-convert-static-mesh-no-asset-echo`
and `E-geometry-deformer-echo-mesh-counts` are the convert-echo and deformer-echo
ergonomics on different verbs. This is the boolean-verb member of the same
partial-echo family, not yet on the board.

## Docs page to improve (if taken docs-only)

`docs/wiki-src/geometry.md` — note that `boolean_subtract`/`boolean_union`/
`boolean_intersection` echo `resultTriangles`/`changed` but **not** the result
vertex count, so a "confirm sensible vertex/triangle count" step needs a follow-up
`geometry.get_mesh_info`, with a `## See also` link to `get_mesh_info`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry` "StonePlanter" hollow-planter build (21 calls incl. 9 wiki-nav, outcome tool_bug, friction:"none", all RPCs first-try clean, no retries/errors/fallbacks; judge filed `B-geometry-convert-static-mesh-no-disk-write` `#2` for the bake-persistence bug). Distinct PROCESS finding: `geometry.boolean_subtract` (and `boolean_union`/`boolean_intersection`, shared success branch `BooleanHandler.cpp:165-188`) is a *partial* count-echoer — it sets `resultTriangles` (`ResultMesh->GetTriangleCount()`, `:137,173`) and `changed` (`:184-185`) but never `resultVertices`/`vertexCount` (no `GetVertexCount` call in the handler, though `ResultMesh` is live at `:135-137`). Story step 4 ("confirm a sensible vertex/triangle count") therefore got the tri half free from the boolean echo (`resultTriangles:336`, `changed:true` — quoted verbatim in the self-report "144->336 tris (changed:true)") but forced a separate `geometry.get_mesh_info` to read the `170` verts (post-bool `170v/336t`), the tri count of which the boolean already reported. Same partial-echo nuance as `E-geometry-deformer-echo-mesh-counts` `#4` (poke echoes triangleCount, omits vertexCount) but on the boolean surface that ticket explicitly excludes (scoped to deformers + `array_*`; its `#8-fix` never touched `BooleanHandler.cpp`). Fix: echo `resultVertices` off `ResultMesh->GetVertexCount()` on the held mesh, folding the vertex/triangle confirmation into the boolean call; docs floor names `docs/wiki-src/geometry.md`. Workaround: `geometry.get_mesh_info` after the boolean for the verts. Dedup: distinct from the deformer-echo, boolean-batch (call-count), boolean-offset (correctness), and convert-disk-write (persistence) tickets.
