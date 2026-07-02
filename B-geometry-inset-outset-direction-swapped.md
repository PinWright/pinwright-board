---
id: B-geometry-inset-outset-direction-swapped
title: "geometry.inset expands faces outward and geometry.outset shrinks them inward — the two ops do the opposite of their name/docs (distance sign is backwards)"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, inset, outset, direction, sign-flip, wrong-result, dynamic-mesh, InsetOutsetFaces]
encounters: 1
lastSeen: 2026-06-30T23:46:26.1606865+03:00
---

# `geometry.inset` outsets (grows the face outward) and `geometry.outset` insets (shrinks it inward) — the distance sign is inverted on both

With the per-face targeting from `B-extrude-inset-empty-selection-whole-mesh`
now live (an optional `faceDirection` builds a real face selection), a
face-targeted inset finally does something — but it moves the geometry in the
**wrong direction**. `geometry.inset` (wiki: *"Inset faces of a dynamic mesh
(shrink inward)"*) pushes the targeted face's boundary **outward** by the inset
distance, and `geometry.outset` (wiki: *"expand outward"*) pulls it **inward**.
The two verbs are effectively swapped.

The op still reports unconditional success (`"Inset applied"`, `changed:true`,
correct `vertexCount`/`triangleCount`), and the count deltas for a real inset
and a real outset are **identical** (both add the same ring of verts/tris), so
the only signal that anything went the wrong way is the `get_mesh_info`
`boundingBox` — which the caller is not required to inspect. A caller who asks
for a recessed inner field (inset to make a frame) instead gets an outward lip,
silently, and builds on it (the seed task baked a "recessed wall panel" whose
frame actually protrudes).

severity rationale: impact=silent-wrong-result-on-normal-path (caller trusts a
result that is the geometric opposite of what was requested, and the response
gives no in-band direction signal) × reach=core modeling verb but not
every-session -> High.

## Root cause (guilty source)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp`
negates the distance for inset and leaves it positive for outset — both
backwards relative to the engine's `FGeometryScriptMeshInsetOutsetFacesOptions::Distance`
convention, where **positive Distance insets inward** (empirically confirmed by
replay, see below):

```cpp
// geometry.inset handler
529:    FGeometryScriptMeshInsetOutsetFacesOptions Options;
530:    Options.Distance = -Distance;  // Negative for inset      <-- WRONG: -Distance OUTSETS
531:    Options.bReproject = true;
...
538:    UGeometryScriptLibrary_MeshModelingFunctions::ApplyMeshInsetOutsetFaces(
539:        Target.Mesh, Options, Selection, nullptr);

// geometry.outset handler
570:    FGeometryScriptMeshInsetOutsetFacesOptions Options;
571:    Options.Distance = Distance;  // Positive for outset       <-- WRONG: +Distance INSETS
572:    Options.bReproject = true;
```

The two comments encode the inverted assumption. The fix is to swap the signs:
inset should pass `Options.Distance = Distance` (positive = inward) and outset
`Options.Distance = -Distance`. (A regression guard should assert via
`get_mesh_info` that a face-targeted inset of distance D does **not** grow the
bounding-box extent, and that outset of D **does** grow it by D.)

## Repro (verbatim, replay-confirmed against mcp__pinwright__call)

```
geometry.create_box {name:"ReplayRecessPanel", width:400, height:300, depth:20}
geometry.get_mesh_info {actorName:"ReplayRecessPanel"}
  -> extent {x:200, y:150, z:10}                                   baseline 400x300x20 slab

geometry.inset {actorName:"ReplayRecessPanel", distance:35, faceDirection:{x:0,y:0,z:1}}
  -> {operation:"inset", vertexCount:12, triangleCount:20, facesSelected:2, changed:true, message:"Inset applied"}
geometry.get_mesh_info {actorName:"ReplayRecessPanel"}
  -> extent {x:235, y:185, z:10}                                   GREW +35 in X and Y == OUTSET (should have stayed 200/150 for an inset)

geometry.create_box {name:"ReplayOutsetCheck", width:400, height:300, depth:20}
geometry.outset {actorName:"ReplayOutsetCheck", distance:35, faceDirection:{x:0,y:0,z:1}}
  -> {operation:"outset", vertexCount:12, triangleCount:20, facesSelected:2, changed:true, message:"Outset applied"}
geometry.get_mesh_info {actorName:"ReplayOutsetCheck"}
  -> extent {x:200, y:150, z:10}                                   UNCHANGED — inner face pulled inward == INSET (an outset should have grown the extent)
```

`inset` (Options.Distance = -35) expanded the footprint outward by 35; `outset`
(Options.Distance = +35) left the outer footprint and pulled the inner face
inward. Positive `Options.Distance` = inset, negative = outset — the opposite of
what both handlers assume.

This is distinct from `B-extrude-inset-empty-selection-whole-mesh` (no
face-targeting + success-on-no-op, IN-REVIEW): that defect made inset a no-op so
the direction was never observable; this is the *direction* of the now-working
op. Distinct from `E-geometry-deformer-echo-mesh-counts` (count echo only).

**Workaround:** call `geometry.outset` when you want an inset (and vice versa),
or negate `distance` mentally; verify direction with `get_mesh_info`'s
`boundingBox` extent before/after — the response message and counts do not
distinguish inset from outset.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed against
  mcp__pinwright__call on a 400x300x20 `create_box` (extent 200/150/10). A
  face-targeted `geometry.inset {distance:35, faceDirection:{0,0,1}}` GREW the
  bbox extent to 235/185/10 (+35 outward = an outset), while
  `geometry.outset {distance:35, faceDirection:{0,0,1}}` left the extent at
  200/150/10 (inner face pulled inward = an inset). The two verbs are swapped.
  Guilty: `MeshOpsHandler.cpp:530` (`Options.Distance = -Distance; // Negative
  for inset`) and `:571` (`Options.Distance = Distance; // Positive for outset`)
  — the engine's `FGeometryScriptMeshInsetOutsetFacesOptions::Distance` insets on
  positive, so both signs are inverted. Op reports success (`changed:true`,
  counts increase identically for either direction); only the bbox extent reveals
  the wrong direction. Surfaced by a "carve a recessed wall panel" task that
  inset the top face to make a frame and got an outward lip instead.
- `#2-fix-sign-swap` `IN-REVIEW` developer — Swapped the inverted
  `Options.Distance` signs in `MeshOpsHandler.cpp`: `geometry.inset` now passes
  `+Distance` (positive Distance insets inward per the engine's `FInsetMeshRegion`
  convention — the boundary is moved toward the region centroid by
  `Distance*InsetDir`) and `geometry.outset` passes `-Distance` (negative grows the
  footprint outward; the engine forces reproject off for the negative case). Updated
  the two misleading comments. Added regression test
  `PinWright.geometry.inset_outset.DirectionMatchesName`
  (`Source/PinWright/Private/Tests/Geometry/TestGeometryInsetOutsetDirection.cpp`)
  that spawns a 400x300x20 box, face-targets the +Z face, and asserts a distance-35
  `geometry.inset` does NOT grow the bounding-box extent (facesSelected >= 1) while a
  distance-35 `geometry.outset` grows it by ~35 — reverting the sign swap fails both
  halves. Files: `Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp`,
  `Source/PinWright/Private/Tests/Geometry/TestGeometryInsetOutsetDirection.cpp`.
