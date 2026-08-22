---
id: B-blossom-pole-ring-degenerate-triangles
title: "sp_blossom_build.py emits a full quad ring at each UV-sphere pole where the radius collapses to zero — 192 degenerate triangles, 192 boundary edges and 192 inconsistent edges in SM_Tree_Radiant_Blossom, and the open boundary then hides a second, real handedness fault from the audit"
status: OPEN
severity: Medium
category: bug
tags: [content, foliage, sp_blossom_build, static-mesh, degenerate-triangles, winding, uv-sphere-pole, audit_static_meshes, masked-defect]
encounters: 1
lastSeen: 2026-08-22T00:00:00Z
---

# A pole that is one vertex, tessellated as if it were eight

`geometry.audit_static_meshes` on `SM_Tree_Radiant_Blossom` returns **three** simultaneous
findings — inconsistent winding, open boundary, and degenerate triangles. That combination is new
on this map: the two hand-wound trees carry a handedness fault and nothing else. This one has a
different root cause, and the three findings are one defect seen three ways.

## The mechanism

`Docs/scripts/foliage/sp_blossom_build.py` builds each blossom cluster as a lobed UV sphere with
`SEG_U = 8` around and `SEG_V = 5` pole to pole (`:145-146`), over **12 clusters**.

The vertex ring is generated for `j` in `0..SEG_V` with `phi = math.pi * j / SEG_V` (`:218`). At
`j = 0` and `j = SEG_V` that gives `phi = 0` and `phi = π`, so `sin(phi) = 0` and the radial term
vanishes: **all 8 vertices of the ring collapse onto the same point.** The pole is one location
stored eight times.

The triangle loop (`:225-232`) then walks every `(j, i)` quad uniformly and emits two triangles per
quad, with no special case for the collapsed rings. At each pole that produces 8 triangles whose
two "ring" corners are the same point:

    12 clusters x 2 poles x 8 = 192

which is exactly the reported count, three times over: **192 degenerate triangles**, **192
zero-length pole edges** presenting as boundary edges, and **192 inconsistent edges**.

## Why the winding finding follows from the same line

`_orient` (`:160-173`) fixes facing by comparing the triangle normal against the outward direction
from an interior point and flipping when the dot product is negative:

```python
out.append([a, b, c] if (n[0]*d[0] + n[1]*d[1] + n[2]*d[2]) >= 0.0 else [a, c, b])
```

On a degenerate triangle the cross product `n` is the **zero vector**, so the dot product is
exactly `0.0`, the `>= 0.0` tie-break is taken, and the raw winding is kept unconditionally —
whatever it happens to be, and without reference to the neighbours it shares edges with. The
inconsistency is not a separate authoring mistake; it is the orientation pass having no signal to
act on, because the triangle has no area to have a normal.

## The finding the audit could not make

**`inverted` requires a closed mesh.** The 192 boundary edges make the closed-only checks
not-applicable on this asset, so `inverted` is not answered at all — and the handedness fault the
two sibling trees carry **is also present here**. It simply cannot be seen while the poles are
open.

That is the part worth carrying forward: **the asset is worse than its audit findings show.**
Fixing the poles will not make the audit go quiet — it will make a fourth finding appear, because a
check that was structurally unanswerable becomes answerable. Anyone reading a post-fix report
should expect that and not treat it as a regression.

It is also a small vindication of the audit's not-applicable accounting: the check did not report
"clean", it reported "not applicable", and those are different answers precisely so this situation
is legible rather than silently reassuring.

## Fix

**A pole fan.** Emit one vertex at each pole and a triangle fan of `SEG_U` triangles connecting it
to the adjacent ring, instead of a full quad ring against a collapsed one. That removes all three
findings at the source: no zero-area triangles, no zero-length edges, no null normals for `_orient`
to tie-break on, and the mesh closes.

Then re-run the audit and expect `inverted` to become answerable; fix the handedness separately if
it fires.

**Do not** patch this by loosening `_orient`'s tie-break or by filtering degenerate triangles after
generation. Both hide the symptom and leave the topology wrong — the mesh would still be open at
both poles of all twelve clusters.

**Scoped follow-up, not this wave.** No other content depends on the fix landing first.

## Note on the script's own self-check

`sp_blossom_build.py` carries `_assert_welded` (`:284`) and a file-header claim that "EVERY CLUSTER
IS WELDED BY CONSTRUCTION, NOT BY LUCK" (`:29`). That check passes on the current output. It
verifies **inter-cluster** welding and says nothing about intra-cluster pole topology, so a green
self-check sat next to 192 degenerate triangles the whole time — worth remembering before quoting
it as evidence of mesh health.

## Severity

**Medium.** By the rubric this is a soft blocker on content rather than a tool defect: the asset
renders and is usable, but its shading is wrong at the poles, it cannot be answered for inversion
at all, and any downstream operation that needs a closed manifold (boolean, distance field,
simplification) has no clean input. There is a workaround only in the sense that the mesh can be
regenerated once the script is fixed — the asset as committed cannot be repaired in place.

**No reach bump.** One asset, from one generator script, and nothing else consumes it. The
generator pattern is worth checking against the other `sp_*` foliage scripts, and if the same
collapsed-pole loop appears in them the reach modifier applies and this should be re-rated.

Not Critical or High: no editor crash, no data-corrupting write, and nothing here is a silent
false-success — the audit reported all three findings loudly and correctly. What it could not
report is called out above as a known blind spot rather than a claim of cleanliness.

## History
- `#1-collapsed-pole-ring` `OPEN` reporter — `geometry.audit_static_meshes` on `SM_Tree_Radiant_Blossom` returns inconsistent winding **plus** open boundary **plus** degenerate triangles simultaneously — a combination not seen on this map before, and distinct from the two hand-wound trees, which carry a handedness fault alone. Diagnosed to `Docs/scripts/foliage/sp_blossom_build.py`: each cluster is a lobed UV sphere with `SEG_U = 8`, `SEG_V = 5` (`:145-146`) over 12 clusters, and the vertex ring runs `j` in `0..SEG_V` with `phi = math.pi * j / SEG_V` (`:218`), so at `j = 0` and `j = SEG_V` the radial term vanishes and all 8 ring vertices collapse onto one point. The triangle loop (`:225-232`) then emits two triangles per quad uniformly with no pole special case, producing 8 zero-area triangles per pole: 12 x 2 x 8 = **192**, matching all three reported counts (192 degenerate triangles, 192 zero-length pole edges as boundary, 192 inconsistent edges). The winding finding follows from the same line rather than being a separate mistake: `_orient` (`:160-173`) flips on a negative dot product against the outward direction, but a degenerate triangle has a zero-vector normal, the dot product is exactly `0.0`, and the `>= 0.0` tie-break keeps the raw winding unconditionally with no reference to neighbours. **The audit could not answer `inverted` at all** — that check requires a closed mesh and the 192 boundary edges make it not-applicable — and the sibling handedness fault IS also present, so the asset is worse than its findings show; expect a fourth finding to appear after the poles are fixed, and do not read that as a regression. Fix is a pole fan: one vertex per pole plus a `SEG_U` triangle fan to the adjacent ring, which removes all three findings at the source and closes the mesh; then re-audit for inversion. Do **not** loosen `_orient`'s tie-break or post-filter degenerate triangles — both hide the symptom and leave both poles of all twelve clusters open. Note the script's own `_assert_welded` (`:284`) and its "welded by construction, not by luck" header claim (`:29`) pass on this output: they verify inter-cluster welding only and say nothing about pole topology, so a green self-check sat beside 192 degenerate triangles. Deduped against the board (1300+ tickets): no existing ticket names this asset, this script, or degenerate-pole topology; the nearest neighbours are unrelated (`B-asset-dump-meta-assettype-degenerate` is a dump-metadata ticket, not geometry). Severity **Medium** per the rubric's soft-blocker band — the asset renders but shades wrongly at the poles, cannot be answered for inversion, and offers no clean input to any operation needing a closed manifold; the only workaround is regeneration after a script fix, since the committed asset cannot be repaired in place. No reach bump: one asset, one generator, nothing else consumes it — but the other `sp_*` foliage scripts should be checked for the same collapsed-pole loop, and if it recurs there the reach modifier applies and this should be re-rated. Not High: nothing here is a silent false-success, the audit reported all three findings correctly, and the one thing it could not report is recorded above as a known blind spot. Scoped follow-up, not this wave.
