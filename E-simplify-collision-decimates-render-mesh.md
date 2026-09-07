---
id: E-simplify-collision-decimates-render-mesh
title: "geometry.simplify_collision ALSO decimates the render mesh (~50% at the default simplificationFactor 0.5), not just the collision hulls — the name + wiki underplay this and don't cross-reference the mesh-preserving hulls-only path (generate_complex_collision maxHullCount=N)"
status: OPEN
severity: Low
category: ergonomic
tags: [geometry, simplify_collision, docs, semantic-mismatch, render-mesh-decimation, collision, discoverability]
encounters: 1
costly: 1
lastSeen: 2026-07-11T04:23:37+03:00
---

# `geometry.simplify_collision` works exactly as documented, but its name/wiki underplay that it decimates the VISIBLE render mesh by default — a caller whose intent is "cut collision cost, keep the mesh intact" is surprised, and the non-destructive alternative isn't discoverable

**This is NOT a defect** — the per-finding Judge replayed the call and confirmed
`simplify_collision` behaves per its documented contract: the wiki summary already
says "Simplify mesh and regenerate collision with fewer hulls", the
`simplificationFactor` param (default 0.5) documents the decimation, the source
(`LODCollisionHandler.cpp`) deliberately simplifies the mesh to `CurrentTris*factor`
THEN regenerates collision, `shapeCount` is a genuine readback (not a hardcoded
echo), and the render-mesh reduction is **disclosed in-band** via the returned
`trianglesBefore`/`trianglesAfter` fields. So it is neither silent nor a lie.

The finding is a **pure discoverability gap**: the method *name* `simplify_collision`
and the wiki wording are not emphatic enough that, at the default
`simplificationFactor=0.5`, the call **halves the actor's visible render mesh** —
and there is no cross-reference to the hulls-only, mesh-preserving alternative that
already exists (`geometry.generate_complex_collision {maxHullCount:N}`). Works, but
non-obvious.

## Why it's non-obvious (ground-truth evidence)

The Attempt (focus `geometry.simplify_collision`, story: cut a boulder prop's
collision down to ~4 hulls *while keeping the visible mesh intact and watertight*)
ran a very smooth trace — 10 real RPCs, zero errors/retries/hangs, `python.execute`
never used, outcome clean. Yet the CallAnalyzer's ground-truth `plan_divergence`
shows the surprise landed anyway:

- The Attempt **read the wiki page** for the method (`wiki/geometry.simplify_collision.md`
  is in the call log) and STILL its plan assumed `simplify_collision` would leave the
  render mesh vertex/triangle counts UNCHANGED.
- Reality: one call `geometry.simplify_collision {actorName:Boulder, targetHullCount:4}`
  (default `simplificationFactor 0.5`) → `shapeCount:4` **and** `trianglesBefore 6348 ->
  trianglesAfter 3174`; the immediate follow-up `geometry.get_mesh_info` confirmed the
  RENDER mesh was halved (`vertexCount 3176 -> 1589`, `triangleCount 6348 -> 3174`),
  bounding box + watertightness preserved.
- The Attempt's own friction note acknowledges the surprise verbatim:
  "simplify_collision inherently halved the render mesh (6348->3174 tris via default
  simplificationFactor 0.5), but it stayed watertight and the silhouette bounding box
  was preserved, so the mesh survived intact."

The task's story explicitly required "the visible mesh stays intact and undamaged"
and "make sure the mesh itself is still clean and watertight afterward" — a caller
with that exact intent, reading the method NAME, would not expect a ~50% render-mesh
decimation. A competent agent read the current wiki and was still surprised, which is
the signal that the wording under-communicates the default destructive effect. It
happened to cost **no extra calls** here (the Attempt was verifying watertightness via
`check_health`/`get_mesh_info` anyway), so this is PROCESS/discoverability overhead,
not agent struggle — but a caller who needs the exact visible geometry preserved would
be silently handed a half-density mesh.

## What the docs should do (downstream wiki process, not a code fix)

Improve the `### geometry.simplify_collision` overlay in
`Plugins\PinWright\Docs\wiki-src\geometry.md` (a.k.a. `docs/wiki-src/geometry.md`) to:

1. State up front that this verb **also simplifies the actor's RENDER mesh** by
   `simplificationFactor` (default 0.5 → ~50% of triangles), not only the collision
   hulls — the name says "collision" but the mesh is decimated too. Note the response
   discloses this via `trianglesBefore`/`trianglesAfter`.
2. Cross-reference the mesh-preserving hulls-only path: to reduce hull count **without
   touching the render mesh**, use `geometry.generate_complex_collision {maxHullCount:N}`
   (e.g. `maxHullCount:4`), which regenerates convex-decomposition collision at N hulls
   and leaves the visible mesh unchanged. This alternative already exists but is not
   discoverable from the `simplify_collision` name/doc.
3. (Optional, for the `simplificationFactor` param line) note that a higher factor
   (toward 1.0) decimates the render mesh less; callers who must preserve exact visible
   geometry should prefer `generate_complex_collision` instead.

No code change is required for the discoverability fix — the behavior is correct and
disclosed. (If a code enhancement is ever wanted, a `simplificationFactor`-of-1.0 /
hulls-only mode that skips render-mesh decimation would remove the trap entirely, but
that is out of scope for this docs ticket and duplicates `generate_complex_collision`.)

## Workaround (available today)

To bring the hull count down to ~N while keeping the render mesh intact, skip
`simplify_collision` and call `geometry.generate_complex_collision {actorName, maxHullCount:N}`
directly (this Attempt's own `generate_complex_collision {maxHullCount:32}` step proves
the verb honors the hull cap — it returned `hullCount:32`; `maxHullCount:4` yields ~4
hulls with the mesh untouched).

## Distinct from related tickets

- `E-geometry-array-radial-merges-in-place` (IN-REVIEW) — same broad symptom FAMILY
  (`semantic-mismatch`: a geometry verb's name implies a narrower scope than its
  behavior), but a **different method and root cause** (array verbs bake N copies into
  the source mesh in place, no actors spawned). Filed separately per the board's
  method/root-cause dedup rule; this ticket seeds the shared `semantic-mismatch` family
  tag so future siblings can aggregate.
- `E-geometry-deformer-echo-mesh-counts` / `E-remesh-uniform-echoes-target-not-achieved`
  — those are *missing/misleading count-echo* response-shape gaps. Here the counts ARE
  echoed correctly (`trianglesBefore`/`trianglesAfter`); the gap is purely that the
  NAME + wiki don't set the expectation that the render mesh is decimated, plus the
  missing cross-reference to the mesh-preserving alternative.
- `B-geometry-convert-static-mesh-drops-collision` — a `convert_to_static_mesh`
  correctness bug, unrelated.

severity rationale: impact=docs/discoverability + naming (works correctly and is
disclosed in-band; no lie, no data loss on the normal path — a caller who reads the
returned trianglesBefore/After sees exactly what happened) → Low × reach=collision
simplification is a specific prop/physics pipeline path, not an every-session verb →
rare, no bump (already at the Low floor) → Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.simplify_collision` "Boulder" physics-prop task (21 calls: 11 wiki-nav + 10 RPCs, all `ok=true` first try, zero errors/retries/hangs, no `python.execute`; outcome clean, `filed_id` empty). The per-finding Judge REPLAYED and disposed it as working-per-contract (wiki says "Simplify mesh and regenerate collision with fewer hulls"; `simplificationFactor` default 0.5 documented; `LODCollisionHandler.cpp` decimates mesh then regenerates collision; `shapeCount` a genuine `GetSimpleCollisionShapeCount` readback; render-mesh reduction disclosed via `trianglesBefore`/`trianglesAfter`, not silent) — so this is filed NOT as a defect but as the pure-discoverability residual the audit rules permit. Ground-truth evidence (CallAnalyzer `plan_divergence`): the Attempt read `wiki/geometry.simplify_collision.md` yet its plan still assumed the render mesh counts would be unchanged; reality: one `simplify_collision {targetHullCount:4}` call (default factor 0.5) returned `shapeCount:4` AND `trianglesBefore 6348 -> trianglesAfter 3174`, and the follow-up `get_mesh_info` confirmed the render mesh halved (`v3176->1589`, `t6348->3174`, bbox + watertightness preserved). The story explicitly required the visible mesh stay intact/undamaged, so the ~50% render-mesh decimation is surprising for that intent even though it cost no extra calls (the Attempt was verifying watertightness anyway). Fix (docs only): improve `docs/wiki-src/geometry.md` `### geometry.simplify_collision` to (a) state the verb also decimates the RENDER mesh by `simplificationFactor` (default 0.5), disclosed via `trianglesBefore`/`trianglesAfter`, and (b) cross-reference the mesh-preserving hulls-only path `geometry.generate_complex_collision {maxHullCount:N}` (proven in-trace: `maxHullCount:32` → `hullCount:32`, mesh untouched). Workaround: use `generate_complex_collision {maxHullCount:N}` when the visible mesh must be preserved. Seeds the `semantic-mismatch` family tag (sibling: `E-geometry-array-radial-merges-in-place`). Severity Low (docs/discoverability, rare pipeline path).
