---
id: E-remesh-uniform-echoes-target-not-achieved
title: "geometry.remesh_uniform echoes the REQUESTED targetTriangleCount but never the ACHIEVED triangleCount — on a budget-targeting verb whose result can diverge ~48% (target 3000 → 4442 tris), so the one field the response carries looks like the answer but is the input, and confirming the actual budget forces a separate get_mesh_info"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [geometry, remesh_uniform, mesh-info, readback, round-trip, response-shape, triangle-count, input-echo, misleading-echo, consistency, docs]
encounters: 1
lastSeen: 2026-07-02T05:47:09+0300
---

# remesh_uniform's success response reports only the requested target, not the count it actually hit — and the requested value can be ~48% off the achieved one

`geometry.remesh_uniform`'s entire purpose is to retopologize a mesh down to a
**triangle budget**, yet its success response carries only the *requested*
target and a static message — never the count it actually produced. Verbatim
from the handler (`MeshOpsHandler.cpp:1333-1336`):

```cpp
TSharedPtr<FJsonObject> Result = MakeShared<FJsonObject>();
Result->SetStringField(TEXT("actorName"), ActorName);
Result->SetNumberField(TEXT("targetTriangleCount"), TargetTriangleCount);  // echoes the INPUT
Ctx.SendSuccess(TEXT("Uniform remesh applied"), Result);
```

So the response is `{actorName, targetTriangleCount:<input>, message:"Uniform
remesh applied"}`. It reads **no** post-op count off `Target.Mesh` — the live
`UDynamicMesh` the handler holds and remeshed one line earlier
(`ApplyUniformRemesh(Target.Mesh, ...)`, `:1328-1329`).

## Why this is worse than a plain no-echo omission

This is a member of the established "the mutator holds the answer but doesn't
carry it, so a working readback verb gets spammed" response-shape family —
`E-geometry-deformer-echo-mesh-counts` (IN-REVIEW; deformers/array/poke),
`E-geometry-boolean-omits-result-vertex-count` (OPEN; boolean ops omit the
vertex half), `E-post-process-setters-no-echo`, `E-lighting-set-ao-exposure-no-echo`,
`E-set-transition-settings-no-echo` — but with a **distinct, more consequential
angle** those don't have:

1. **The one field it carries is actively mistakable for the answer.** The
   deformer family returns `{actorName}` (or `{actorName, angle}`) — nothing that
   looks like a count, so nothing to be misled by; you just know you must read
   back. remesh_uniform returns a **triangle-count-named field**
   (`targetTriangleCount`) holding the *requested* value. A caller reading the
   response to answer "did the retopo hit my budget?" sees `targetTriangleCount:3000`
   and can reasonably conclude the mesh is now ~3000 tris — the classic
   input-echo-mistaken-for-result trap.
2. **The divergence is large and load-bearing.** A deformer usually preserves the
   count, so its omission is mostly a sanity probe. Uniform remesh *re-tessellates
   the whole surface toward a target it only approximates*: this task asked for
   3000 and got **4442** (a ~48% overshoot). Hitting-the-number IS the verb's
   success criterion, and the number the response shows is off by half.
3. **It is a topology op, not a deformer** — sibling to `subdivide`
   (`{originalTriangles, subdividedTriangles}`) and `simplify_mesh`
   (`{originalTriangles, simplifiedTriangles, reductionPercent}`) in the very same
   `MeshOpsHandler.cpp`, both of which already echo their realized post-op counts.
   remesh_uniform is the intra-file odd-one-out: the budget-targeting verb is the
   one that hides whether it hit the budget.

## What it should do

Echo the **achieved** triangle count (and ideally `vertexCount`) inline off the
live `Target.Mesh` the handler already holds — one `Target.Mesh->GetTriangleCount()`
— e.g. `{actorName, targetTriangleCount:3000, resultTriangleCount:4442, changed:true}`,
mirroring the `resultTriangles` that `boolean_union` already returns and the
`subdividedTriangles`/`simplifiedTriangles` its topology-op siblings return. Name
it so it cannot be confused with the echoed input (`resultTriangleCount`, not a
second `triangleCount` sitting next to `targetTriangleCount`). This folds the
"did the retopo land near my budget?" confirmation into the remesh call and
removes the forced follow-up `get_mesh_info`.

Secondary (docs floor): `Plugins\PinWright\Docs\wiki-src\geometry.md` (the
`### geometry.remesh_uniform` overlay section) documents only
`targetTriangleCount (integer, optional): Target triangle count (default 5000)`
with no Notes/overlay warning that uniform remesh **approximates** the target and
the achieved count can diverge substantially, nor that the response echoes the
*requested* target rather than the achieved count (so confirming the real budget
needs a `geometry.get_mesh_info`). One line there sets caller expectations even if
the response fix is deferred.

**Workaround:** call `geometry.get_mesh_info {actorName}` after the remesh to read
the real `triangleCount` (this task's `4442`) — do **not** trust the echoed
`targetTriangleCount` as the achieved count.

## Friction evidence (this task — focus `geometry.remesh_uniform`, namespace `geometry`, game-ready-hero-prop build, outcome clean, friction:"none")

Struggle audit of the "HeroProp" game-ready-prop build: `create_sphere` +
`create_box` → `boolean_union` (N0=9642 messy seam tris) → `remesh_uniform
target=3000` → `get_mesh_info` → `recalculate_normals` → `generate_collision` →
`convert_to_static_mesh` → `asset.exists`. 12 calls (2 wiki-nav + 10 RPCs), all
`ok=true` first try, no retries/errors/`python.execute`; the judge filed nothing.
The story's explicit closing ask was a first-class read-back: *"Confirm the retopo
actually hit close to the triangle budget I asked for."* The call log shows the
remesh→readback pair the response shape forced:

- `geometry.remesh_uniform {actorName:HeroProp, targetTriangleCount:3000}` → ok;
  response `{"actorName":"HeroProp","targetTriangleCount":3000,"message":"Uniform
  remesh applied"}` — the only count-shaped field is the echoed **input** 3000.
- `geometry.get_mesh_info {HeroProp}` → `triangleCount:4442` (extent 69.99→69.93
  preserved) — fired to read the **achieved** 4442 that the story's budget-confirm
  criterion required and the remesh response could not supply (and would have
  actively misrepresented as 3000).

The agent recovered cleanly because it happened to want that `get_mesh_info` for
the silhouette check anyway, and the mcp-test-workflow's own success check had
pre-warned it that remesh only approximates — so it did not *experience* this as
friction (self-report friction:"none", CallAnalyzer confidence: low). But that is
task-specific luck: a caller whose sole verify step is "confirm the budget" is
forced into a separate `get_mesh_info`, and a caller who trusts the echoed
`targetTriangleCount` gets the wrong answer on the one thing the verb exists to do.
This is the PROCESS/response-shape gap, verified in source, not agent struggle.

## Dedup

No existing ticket covers remesh_uniform (ripgrep `remesh`/`retopo`/`remesh_uniform`
across OPEN+closed → only the unrelated `B-skeleton-copy-weights-noop-zero-fill`).
Distinct from the family siblings:
- `E-geometry-deformer-echo-mesh-counts` (IN-REVIEW) — its `#8-fix` wired
  `SetMeshCountFields` onto the *deformers* + `array_*` + `poke` in
  `MeshOpsHandler.cpp`/`GeometryTransformHandler.cpp`; its scope and regression
  test (`PinWright.geometry.deformers.EchoMeshCounts`) **never include
  remesh_uniform**, so the remesh handler still ships the bare input-echo. Filing
  separately (rather than appending to a fix-landed IN-REVIEW ticket) matches how
  the board already split `E-geometry-boolean-omits-result-vertex-count` out for
  the same reason ("its fix never touched that handler").
- `E-geometry-boolean-omits-result-vertex-count` (OPEN) — boolean ops, and a
  *partial* echoer (has `resultTriangles`, omits the vertex count); remesh omits
  the achieved *triangle* count entirely and echoes the input in its place.
- `B-mesh-info-empty-nonfinite-json` — a `get_mesh_info` correctness bug, not a
  mutator response-shape gap.

severity rationale: impact=a budget-targeting verb reports only the requested
target and no achieved count, and the single count-shaped field it does carry is
mistakable for the result on the exact "did it hit the budget?" success criterion
where the divergence is ~48% — above the pure-omission siblings' "forces an extra
Read" (Low), below High's fabricated-lie (the field name is an honest input label,
not a faked achieved count) → Medium × reach=the retopo step is a specific
game-ready-prop pipeline path, not every-session → no bump → Medium

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.remesh_uniform` "HeroProp" game-ready-prop build (12 calls: 2 wiki-nav + 10 RPCs, all ok=true first try, no retries/errors/fallbacks; outcome clean, friction:"none"; CallAnalyzer confidence low; judge filed nothing). PROCESS/response-shape finding, verified in source: `geometry.remesh_uniform` (`MeshOpsHandler.cpp:1307-1338`) echoes only the requested `targetTriangleCount` (`:1335`) + `"Uniform remesh applied"` and reads NO post-op count off the live `Target.Mesh` it just remeshed (`:1328-1329`). On a budget-targeting retopo verb the achieved count diverges materially — this task's `target=3000` produced `triangleCount=4442` (~48% over) — so the one count-shaped field the response carries is the *input*, mistakable for the achieved result on the story's explicit closing criterion ("confirm the retopo actually hit close to the triangle budget"). Response captured verbatim: `{"actorName":"HeroProp","targetTriangleCount":3000,"message":"Uniform remesh applied"}`; the immediately-following `geometry.get_mesh_info` had to read the real `4442`. Distinct from the no-echo family siblings because it *actively echoes a misleading input* (deformers return bare `{actorName}`; nothing to mislead) and it is a topology op whose in-file siblings `subdivide`/`simplify_mesh` already echo their realized counts. Fix: echo `resultTriangleCount` (and `vertexCount`) off `Target.Mesh->GetTriangleCount()`, mirroring `boolean_union`'s `resultTriangles`; docs floor: add a Notes line to `docs/wiki-src/geometry.md` `### geometry.remesh_uniform` that remesh approximates the target and the response reports the request, not the achieved count. Workaround: `geometry.get_mesh_info` after the remesh; don't trust the echoed target. Not covered by `E-geometry-deformer-echo-mesh-counts` (IN-REVIEW; its `#8-fix`/test exclude remesh) or `E-geometry-boolean-omits-result-vertex-count` (boolean, partial-echo); filed separately per the board's own boolean-split precedent. Severity Medium (mislead-on-budget-verb on a specific retopo pipeline path).
- `#2-fix` `IN-REVIEW` developer — GO. Verified the defect live in current HEAD: `geometry.remesh_uniform` (`Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp:1333-1336`) echoed only the input `targetTriangleCount` + `"Uniform remesh applied"` and read no post-op count off the `Target.Mesh` it remeshed at `:1328-1329`. Confirmed intra-file odd-one-out (`simplify_mesh` echoes originalTriangles/simplifiedTriangles/reductionPercent `:257-259`; `subdivide` echoes originalTriangles/subdividedTriangles `:334-335`; `boolean_union` echoes `resultTriangles` `BooleanHandler.cpp:173`) and NOT covered by the deformer fix (its test `PinWright.geometry.deformers.EchoMeshCounts` excludes remesh_uniform; no other ticket/test touches it). Fix: added one `GeometryUtils::SetMeshCountFields(Target.Mesh, Result)` after the remesh — the established helper this file already uses ~13× (`:648`…`:1207`) — so the response now also carries the ACHIEVED `vertexCount`/`triangleCount` read off the just-remeshed mesh, matching the keys `geometry.get_mesh_info` returns. Chose the helper's `triangleCount` key over a bespoke `resultTriangleCount`: it is a distinct key from `targetTriangleCount` (un-confusable with the input echo, satisfying the ticket's naming concern) and consistent with get_mesh_info + the deformer siblings, per the correctness/adversarial lenses' guidance. Files: `MeshOpsHandler.cpp` (remesh_uniform handler). Regression test: `PinWright.geometry.remesh_uniform.EchoMeshCounts` (`Source/PinWright/Private/Tests/Geometry/TestGeometryRemeshUniformEchoesMeshCounts.cpp`) — spawns a real DynamicMeshActor via `geometry.create_box` (in-code fixture, no external asset), routes `geometry.remesh_uniform target=2000` through the real dispatcher, and asserts the success result carries the echoed input `targetTriangleCount` AND a non-zero achieved `vertexCount` + `triangleCount`; reverting the fix drops the achieved counts and fails it. Docs floor (the secondary "add a Notes line to geometry.md" item) left for a follow-up — geometry.md has no `### geometry.remesh_uniform` section at all (the quoted param line is auto-generated from the `RPC_PARAM_OPT` at `:1310`), so it is a docs-add, not the response-shape fix this ticket's primary ask targets.
