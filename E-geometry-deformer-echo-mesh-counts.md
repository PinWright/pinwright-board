---
id: E-geometry-deformer-echo-mesh-counts
title: "geometry deformers (twist/taper/bend/noise_deform/smooth/relax/stretch/spherify/cylindrify/bevel/shell/chamfer) and array_radial/array_linear don't echo vertexCount/triangleCount; poke echoes only triangleCount — forces a separate get_mesh_info readback, while subdivide/simplify/extrude/inset already echo counts"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [geometry, twist, taper, bend, deformer, mesh-info, readback, round-trip, response-shape, consistency]
---

# Geometry deformer responses omit the post-op vertex/triangle counts, so confirming a deform "didn't collapse the mesh" always costs an extra get_mesh_info call

The geometry deformers — `geometry.twist`, `geometry.taper`, `geometry.bend`,
and their siblings `noise_deform`, `smooth`, `relax`, `stretch`, `spherify`,
`cylindrify`, `bevel`, `shell`, `chamfer` — plus the array verbs
`geometry.array_radial` / `geometry.array_linear` (absorbed here, see `#5`)
return only `{actorName}` plus (sometimes) an echo of the input parameter.
They do **not** report the resulting vertex or triangle count, even though that
count is the single most common "did it work / did it collapse?" signal after a
mutate and is trivially available on the same `Target.Mesh` the handler already
holds (for the array verbs the count *multiplies by `count`*, making the post-op
count even more load-bearing). `geometry.poke` is a *partial* echoer — it
returns `triangleCount`/`originalTriangles` but omits `vertexCount` (see `#4`).

**Already count-echoing — out of scope here (do NOT re-touch):** the face ops
`geometry.extrude` (MeshOpsHandler.cpp:486), `geometry.inset` (:527),
`geometry.outset` (:568) and `geometry.offset_faces` (:643) now echo
`vertexCount`/`triangleCount`/`facesSelected`/`changed` via the shared
`SnapshotMeshCounts`/`ReportMeshChange` helpers (MeshOpsHandler.cpp:100-166),
added by `B-extrude-inset-empty-selection-whole-mesh` `#3-fix`. They were on
this ticket's omitter list at `#1` but are now fixed.

This is an **intra-namespace inconsistency**, not a missing capability. The
topology-changing ops in the very same `MeshOpsHandler.cpp` already echo
before/after counts:
- `geometry.subdivide` → `{originalTriangles, subdividedTriangles}` (MeshOpsHandler.cpp:334-335)
- `geometry.simplify_mesh` → `{originalTriangles, simplifiedTriangles, reductionPercent}` (MeshOpsHandler.cpp:257-259)
- `geometry.append_vertex` / `delete_vertex` → `{vertexCount}` (MeshInfoHandler.cpp:255,299)
- `geometry.append_triangle` / `delete_triangle` → `{triangleCount}` (MeshInfoHandler.cpp:350,394)

But `geometry.twist` returns only `{actorName, angle}` (MeshOpsHandler.cpp:773-775),
`geometry.bend` only `{actorName, angle}` (MeshOpsHandler.cpp:740-741), and
`geometry.taper` only `{actorName}` (MeshOpsHandler.cpp:808-809) — it doesn't
even echo its own `flareX`/`flareY`/`extent` inputs. The counts the agent needs
are exactly what `geometry.get_mesh_info` reads off the same mesh
(`vertexCount`/`triangleCount`, MeshInfoHandler.cpp:103-104), so every
"confirm the deform is still valid" step degrades into a second round-trip.

## What it should do

Deformer success responses should echo `vertexCount` and `triangleCount`
(matching the keys `get_mesh_info` uses), mirroring how `subdivide`/`simplify`
already report their post-op topology. A deform doesn't change the vertex/tri
count in the common case, but reporting it lets the caller confirm
non-collapse in the same call instead of firing a follow-up `get_mesh_info`.
Cheap, since the handler already holds `Target.Mesh`; both values are one
`GetVertexCount` / `GetTriangleCount` call away. `poke` should additionally
echo `vertexCount` next to the `triangleCount` it already returns.

**Fix:** reuse the existing in-file `SnapshotMeshCounts` helper
(MeshOpsHandler.cpp:100-108) — which already pairs the `MeshQueryFunctions`
`GetVertexCount` accessor with `UDynamicMesh::GetTriangleCount()` (the library
triangle-count accessor is unavailable in UE 5.7) — to set `vertexCount` /
`triangleCount` on each deformer's success result. The array verbs in
`GeometryTransformHandler.cpp` already compute `TriCountBefore`; they need a
`GetVertexCount` + post-op count echo. Do NOT reinvent a parallel snapshot
mechanism, and do NOT re-touch extrude/inset/outset/offset_faces.

**Workaround:** call `geometry.get_mesh_info {actorName}` after each deform to
read `vertexCount`/`triangleCount` (which is exactly what this task had to do
three times).

## Friction evidence (this task — geometry.twist "TwistSpire" column, 17 calls, outcome clean)

The story (step 5) explicitly asked to "check the mesh info after each
deformation … confirm the mesh is still valid (non-zero vertices and
triangles)." Because no deformer echoes counts, the agent was forced into a
strict deform→readback ping-pong — **3 extra `get_mesh_info` round-trips** that
a count-echo in the deform response would have eliminated:

- `geometry.twist {angle:120, extent:150}` → (no counts) → `geometry.get_mesh_info` (post-twist readback)
- `geometry.taper {flareX:70, flareY:70, extent:150}` → (no counts) → `geometry.get_mesh_info` (post-taper)
- `geometry.bend {angle:15, extent:150}` → (no counts) → `geometry.get_mesh_info` (post-bend)

(plus a baseline and a post-subdivide `get_mesh_info`, and a final
success-check readback — 6 `get_mesh_info` calls total interleaved through a
12-step build). The task self-report and `friction:"none"` confirm nothing
errored; this is pure PROCESS overhead, not an outcome bug — the seed
`geometry.twist` landed `clean` in the ledger. The 3 deform→readback pairs are
the same "N calls where the mutator could carry the answer" shape, recurring
once per deformer; it scales linearly with how many deforms a column/prop build
chains.

Distinct from the existing readback tickets `E-get-ai-info-no-perception-readback`
and `E-game-framework-info-not-asset-readback`: those are about a *thin
`get_*_info` read verb* that can't surface its namespace's writes. Here the read
verb (`get_mesh_info`) is fine — the gap is that the *mutators* don't carry the
trivially-available count, so a working read verb gets spammed once per deform.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.twist` "TwistSpire" stone-column task (17 calls, outcome clean, friction:"none", judge filed nothing). PROCESS finding: deformers (`twist`/`taper`/`bend` and siblings) return only `{actorName}`(+input echo) and omit `vertexCount`/`triangleCount`, while same-file `subdivide`/`simplify_mesh`/`append_*`/`delete_*` already echo post-op counts. The story's "check mesh info after each deformation" step therefore forced 3 separate `geometry.get_mesh_info` readbacks (post-twist/taper/bend) that an inline count-echo would eliminate. Evidence: MeshOpsHandler.cpp twist(645-646)/bend(611-612)/taper(679-680) vs subdivide(254-255)/simplify(177-179); counts are the same `GetVertexCount`/`GetTriangleCount` get_mesh_info already reads (MeshInfoHandler.cpp:102-103). Fix: have deformer success responses echo `vertexCount`/`triangleCount`. Workaround: call `geometry.get_mesh_info` after each deform.
- `#2-recurrence-templecolumn` `OPEN` reporter — Recurrence (second task, same shape). Struggle audit of the `geometry.twist` "TempleColumn_01" twisted-column task (11 calls, outcome clean). The story (steps 2/4/5) asked to read mesh info before, after the twist, and after the taper to confirm topology was preserved; because neither `twist` nor `taper` echoes counts, the agent ran the same deform→readback ping-pong — `twist {angle:120, extent:150}` → `get_mesh_info` (post-twist), `taper {flareX:70, flareY:70, extent:150}` → `get_mesh_info` (post-taper) — plus a baseline `get_mesh_info`: **3 `get_mesh_info` calls** the same count-echo would have folded into the deform responses (and 2 separate `actor.get_bounding_box` calls on top, since get_mesh_info also returns no bbox — see also the bbox gap). All 74v/144t, confirming the readbacks carried no surprise. Same fix and workaround as #1; this is the second independent task exhibiting the deformer→readback overhead, confirming it scales once per deform in any column/prop build.
- `#3-recurrence-bevel-pedestalblock` `OPEN` reporter — Recurrence (third task) and the **first direct evidence for `geometry.bevel`** itself (the method this ticket has listed in its deformer set since #1 but only had twist/taper evidence for). Struggle audit of the `geometry.bevel` "PedestalBlock" carved-pedestal task (8 calls, friction:"none", all 7 RPCs first-try clean, judge filed nothing). Story step 3 beveled the box, then step 4 *explicitly* required "Read the mesh info again with geometry.get_mesh_info and confirm the bevel actually changed the topology (vertex and triangle counts must be higher than the baseline)". Because `geometry.bevel` returns only `{actorName}` and no counts, the verify-the-deform-grew-geometry intent forced the exact deform→readback pair: `create_box {w120 d80 h100 segs2}` → `get_mesh_info` (baseline 8v/12t) → `bevel {dist6 subdiv2}` → `get_mesh_info` (post-bevel 24v/44t). A `vertexCount`/`triangleCount` echo on the bevel response would have let step 3 *and* step 4 collapse into one call — the "did the bevel add geometry?" check is precisely the non-collapse signal #1 argues belongs inline, and here the story's higher-than-baseline assertion makes the count the literal success criterion, not just a sanity probe. (Outcome was agent_fail per the judge's separate lens, but the self-report and friction note agree every call succeeded and counts rose strictly — this entry is the PROCESS overhead only, identical fix/workaround to #1.) Confirms the readback cost recurs across the whole deformer family, not just twist/taper.
- `#5-absorbs-array-verb-count-echo` `OPEN` developer — Scope absorption from `E-geometry-array-radial-merges-in-place`. That ticket's adversarial review found its *count-echo* half (echo post-op `vertexCount`/`triangleCount` on the array responses) to be a member of THIS family-wide gap, not a standalone item, so the array-verb count echo is now owned here. `geometry.array_radial` and `geometry.array_linear` (GeometryTransformHandler.cpp:308-312, 215-218) currently echo only `{actorName, count[, angle]}` and report neither `vertexCount` nor `triangleCount`, even though the array operation multiplies the source mesh's geometry by `count` in place — exactly the post-op count signal this ticket argues the whole mutator family should carry (matching `subdivide`/`simplify_mesh`). Same fix as #1 (echo `vertexCount`/`triangleCount` off `Target.Mesh->GetVertexCount()`/`GetTriangleCount()` on the handle the handler already holds), extended to the two array verbs. (The array tickets' OTHER half — the name/doc-vs-behavior semantic mismatch — stays in `E-geometry-array-radial-merges-in-place`, which has been reworded to docs-only and is in IN-REVIEW.)
- `#6-recurrence-shell-noise-stonewell` `OPEN` reporter — Recurrence (fifth task) and the **first direct evidence for `geometry.shell` and `geometry.noise_deform`** (both listed in this ticket's deformer set since #1 but only ever backed by twist/taper/bevel/poke evidence). Struggle audit of the `geometry.shell`/`noise_deform` "StoneWell" build (15 calls, outcome clean, friction:"none", all RPCs first-try clean, judge filed nothing). The story chained `create_pipe → bevel → shell → noise_deform → recalculate_normals → unwrap_uv → pack_uv_islands → generate_collision → convert_to_static_mesh`, with the closing step "Check the mesh info (vertex/triangle counts) to confirm it ended up at a sane density." Because no deformer in the chain echoes counts, the agent fired a `geometry.get_mesh_info` readback right after `shell` (and a baseline after `create_pipe`, and a final pre-convert read) — the same deform→readback ping-pong as #1-#4. The self-report literally tracks the shell-driven jump ("144/236 tris initially, 672 verts / 1336 tris after bevel+shell") — counts that `bevel`/`shell` should have carried inline, sourced from the `Target.Mesh->GetVertexCount()`/`GetTriangleCount()` the handler already holds. Note `recalculate_normals` and `noise_deform` ran *without* an interleaved readback here (the agent batched the density check at the shell boundary rather than after every op), so the overhead surfaced as a single post-shell read rather than the strict per-op ping-pong of #1 — but the gap is identical: the mutator can't report the topology, so a separate read verb is spammed to confirm "sane density." Same fix/workaround as #1; confirms the cost recurs across `shell`/`noise_deform`, not just the twist/taper/bevel/poke set already evidenced.
- `#7-additional-bevel-industrialpipe` `OPEN` reporter — Additional evidence (sixth task) and the **second direct `geometry.bevel` recurrence** (reinforcing `#3-recurrence-bevel-pedestalblock`). Struggle audit of the `geometry.bevel` "IndustrialPipe_01" procedural-pipe-prop build (10 RPCs, outcome clean, friction:"none", all first-try clean, judge filed nothing). The story chained `create_pipe → get_mesh_info(baseline) → bevel → get_mesh_info(post) → recalculate_normals → auto_uv → generate_collision → convert_to_static_mesh → asset.exists → asset.get_metadata`, with step 4 *explicitly* requiring "Read the mesh info again so I can confirm the bevel actually added geometry (triangle count should go up versus the baseline)" — making the post-op count the literal success criterion, identical to `#3`. Replay-confirmed verbatim: `geometry.bevel {actorName:IndustrialPipe_01, distance:4, subdivisions:2}` returned only `{"actorName":"IndustrialPipe_01","distance":4,"message":"Bevel applied"}` (no `vertexCount`/`triangleCount`), forcing the immediately-following `geometry.get_mesh_info` to read `vertexCount:576, triangleCount:1084` (baseline was `320v/572t`). A count-echo on the bevel response would have folded the baseline-vs-post comparison into the create+bevel calls and eliminated the post-bevel readback. The bevel functioned correctly (counts rose strictly, 572→1084 tris), so this is PROCESS overhead only — same fix/workaround as #1, second independent bevel-specific datapoint. (Note: the same task's `auto_uv` rejected a `method:"XAtlas"` arg with a clean `[UNKNOWN_PARAMS]` error — correct behavior, `auto_uv` takes only `actorName`; not a finding.)
- `#8-mesh-repair-verbs-no-count-forced-source-dive` `OPEN` reporter — Recurrence (seventh task) extending this gap to the **mesh-repair family** (`geometry.remove_degenerates`, `geometry.merge_vertices`) — the first datapoint where the missing count didn't just cost an extra readback but made a no-op **undiagnosable in-band**, forcing a dual C++ source-dive. Struggle audit of the `geometry.remove_degenerates` "PropCleanupSphere" mesh-cleanup task (13 calls, outcome tool_bug; the load-bearing `merge_vertices` defect is `B-merge-vertices-welds-edges-not-vertices`). PROCESS finding: `geometry.remove_degenerates` (MeshOpsHandler.cpp:1213-1215) returns only `{actorName}` + `"Degenerate geometry removed"` — **no removed-triangle/vertex count**, even though *removing* triangles is the verb's entire purpose and the count is the literal success signal the story demanded ("confirm the cleanup actually reduced the triangle/vertex count"). Because the response is an unqualified success with no delta, the agent had no in-band way to tell the call no-op'd from the call working, so it had to crack open BOTH the PinWright handler (MeshOpsHandler.cpp) AND the UE engine GeometryScript source (MeshRepairFunctions.cpp) to discover `RepairMeshDegenerateGeometry`'s QEM pass left the count unchanged — friction quoted verbatim: *"Needing to crack open plugin + engine C++ to understand why two success-reporting verbs were no-ops is itself a discoverability/correctness gap."* The agent was then forced into the same separate `geometry.get_mesh_info` readback the deformer entries describe (final read still 728v/1452t, identical to pre-cleanup). `merge_vertices` (MeshOpsHandler.cpp:1292) *does* echo `verticesBefore`/`verticesAfter`/`merged`, which is exactly why its no-op was detectable (`merged:0`) — direct in-ticket proof that the count-echo is the diagnosability mechanism: the verb that echoed counts exposed its own no-op, the verb that didn't (`remove_degenerates`) hid it. Narrowest fix per #1: have `remove_degenerates` echo `triangleCount`/`vertexCount` (and ideally a `removed` delta) off the `Target.Mesh->GetTriangleCount()`/`GetVertexCount()` handle it already holds, mirroring `subdivide`/`simplify_mesh`. This is the first mesh-*repair*-verb evidence (prior entries are all deformer/array verbs); confirms the omit-count cost recurs into the cleanup family and that there it escalates from "extra readback" to "must read engine source."
- `#4-partial-echo-poke-omits-vertexcount` `OPEN` reporter — Recurrence (fourth task) with a **distinct partial-echo nuance**: `geometry.poke` is a *partial* count-echoer, not a total omitter. Struggle audit of the `geometry.poke` "CrystalShard" faceted-crystal prop task (14 calls incl. 7 wiki-nav, friction:"none", all 7 RPCs first-try clean, judge filed nothing). `geometry.poke` (MeshOpsHandler.cpp:1348-1353) already echoes `triangleCount`+`originalTriangles` (the self-report's "48->384 tris" came free from the poke response, like subdivide/simplify) — so it is on the *good* side of this ticket for tris. But it **omits `vertexCount`**, even though poke (offset_faces + PN-tessellate) is precisely the op that explodes the vertex count (here 26→196, a 7.5x jump, larger than the tri jump). The story's final success criterion (steps 6 + closing "compare the before/after stats") *explicitly* required reporting and comparing the final **vertex** count, which poke's response could not satisfy, forcing a final `geometry.get_mesh_info` (196v/384t) purely to read the verts the poke response should have carried alongside the tris it already does. Net readbacks for the build: baseline `get_mesh_info` (8v/12t), pre-poke `get_mesh_info` (26v/48t after subdivide), and the final `get_mesh_info` (196v/384t) — the last forced solely by the missing `vertexCount` on poke. Fix is the narrowest possible extension of #1: poke (and any partial-echoer) should echo `vertexCount` next to the `triangleCount`/`originalTriangles` it already returns — a single `Target.Mesh->GetVertexCount()` on the handle it already holds. Confirms the cost recurs even for mutators that echo *some* counts, when the echoed set doesn't match the dimension the story asserts.
- `#8-fix` `IN-REVIEW` developer — Reworded (dropped extrude/inset/outset/offset_faces from the omitter list — they already echo counts via the `SnapshotMeshCounts`/`ReportMeshChange` helpers added by `B-extrude-inset-empty-selection-whole-mesh` `#3-fix`; refreshed the drifted line citations; pointed the Fix at reusing the existing helpers) and implemented. Added a one-line `SetMeshCountFields(UDynamicMesh*, Result)` helper next to the existing `SnapshotMeshCounts` in `MeshOpsHandler.cpp` (it sets `vertexCount`/`triangleCount` from the same snapshot, matching `get_mesh_info`'s keys) and wired it onto every omitting deformer's success result: `bevel`, `shell`, `chamfer`, `bend`, `twist`, `taper`, `noise_deform`, `smooth`, `relax`, `stretch`, `spherify`, `cylindrify` (taper also now echoes its own `flareX`/`flareY`/`extent` inputs, which it previously dropped). `poke` gains the missing `vertexCount` next to the `triangleCount`/`originalTriangles` it already returned (the #4 partial-echo nuance). The two array verbs in `GeometryTransformHandler.cpp` — `array_linear` and `array_radial` — now echo `vertexCount` (`GetVertexCount`) + `triangleCount` (`GetTriangleCount`) off the post-merge mesh. Did NOT re-touch extrude/inset/outset/offset_faces (already done). Files: `Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp`, `Source/PinWright/Private/Handlers/Geometry/GeometryTransformHandler.cpp`. Test: new `Source/PinWright/Private/Tests/Geometry/TestGeometryDeformerEchoesMeshCounts.cpp` (`PinWright.geometry.deformers.EchoMeshCounts`) spawns a real DynamicMeshActor via `geometry.create_box` then routes twist/taper/bend/smooth/relax/stretch/noise_deform/spherify/bevel/shell/poke/array_linear/array_radial through the real dispatcher and asserts each success result carries a non-zero `vertexCount` AND `triangleCount` — reverting the echo drops the fields and fails it.
