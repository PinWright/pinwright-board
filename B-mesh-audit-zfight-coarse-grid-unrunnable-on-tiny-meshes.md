---
id: B-mesh-audit-zfight-coarse-grid-unrunnable-on-tiny-meshes
title: "geometry.audit_static_meshes z_fighting goes UNRUNNABLE on 160-triangle meshes because ONE large triangle blows the 256 coarse-grid reference limit — 5 of 18 kit meshes unmeasurable and `pass` false for a reason no finding explains"
status: OPEN
severity: Medium
category: bug
tags: [geometry, audit_static_meshes, z-fighting, unrunnable, coarse-grid, environment-review, blockout-meshes]
encounters: 1
lastSeen: 2026-09-02T22:20:00Z
---

# `z_fighting` cannot run on the smallest possible meshes

`geometry.audit_static_meshes` on an 18-mesh environment kit returned
`z_fighting: flagged 2, clean 11, unrunnable 5`. Every unrunnable mesh carries the
same message, differing only in the triangle index:

> Large triangle 28 inspected more than the bounded coarse-grid reference limit
> (256); the detector refused to scan a dense fallback bucket or report a partial
> clean result.

The five: `SM_ENV_WallPanel` (triangle 28), `SM_ENV_Container20` (6),
`SM_ENV_Crate` (6), `SM_ENV_RoofPanel` (8), `SM_ENV_Truck` ("a fighting pair could
not be reconstructed for region area union").

These are not dense meshes. Measured with `static_mesh.describe` in the same
session:

| Mesh | Triangles | Vertices | z_fighting |
|---|---|---|---|
| `SM_ENV_WallPanel` | 160 | 314 | UNRUNNABLE |
| `SM_ENV_Container20` | 562 | 1126 | UNRUNNABLE |
| `SM_ENV_Truck` | 486 | 784 | UNRUNNABLE |
| `SM_ENV_SandbagWall` | — | — | flagged: 14 regions / 672 pairs |
| `SM_ENV_WindowFrame` | — | — | flagged: 2 regions / 8 pairs |

A 160-triangle box with a recessed panel is about the simplest subject the check
can ever be handed. The blocker is not mesh density — it is that **one triangle is
large relative to the coarse grid**, so its cell-occupancy count crosses 256 and
the whole mesh is abandoned. That inverts the expected relationship: the coarser
and blockier the mesh, the more likely the check refuses to run, and a hand-built
modular kit (large flat wall/floor/roof quads) is exactly the content that trips it
most often.

## What I called

    call({method: "geometry.audit_static_meshes",
          args: {folder: "/Game/FPS/Env/Meshes"}})

Response (`structuredContent`): `pass: false`, `failOn: "error"`, ten checks; nine
report `flagged: 0, unrunnable: 0`; `z_fighting` reports `flagged: 2, clean: 11,
unrunnable: 5`. Findings list carries seven `z_fighting` rows at severity
`warning`, five of them the unrunnable message above.

## What happened vs what I expected

`pass` is `false`, and the documented rule is *"pass = no finding at or above
failOn, AND zero unrunnable checks, AND the sweep was not truncated."* So the
folder fails on the strength of five unrunnable meshes even though `failOn` is
`error` and **there is not a single error-severity finding in the whole sweep**.
The caller is told the kit failed, and the only way to learn that the failure is
"the detector gave up" rather than "the geometry is wrong" is to read every
finding message. Worse, the refusal is silent about scale: nothing in the response
says how large "large" is, what the grid cell size was, or what the caller could
change (subdivide? pass a threshold?) to get an answer.

Expected either:

1. the check runs — subdivide the offending triangle, or fall back to a pairwise
   O(n²) test which on 160 triangles is 12,720 comparisons and free; or
2. the refusal is actionable — report the grid cell size, the triangle's projected
   area, and a `thresholds`-style knob to raise the reference limit, the way
   `level.audit` echoes every threshold it compared against.

The current behaviour gives neither: 28 % of a trivially small kit is
unmeasurable, with no lever and no number.

## Why it matters here

This was hit during a blind-A/B environment critique where intra-mesh z-fighting
on the modular kit was one of the technical checks being scored.
`render.detect_z_fighting` from one camera returned a clean
`affectedPixels: 0`, but that is one pose in a 150 × 150 m level and cannot
substitute for a per-asset sweep. With five of eighteen kit meshes unrunnable, the
per-asset half of the question could not be answered at all, and the review had to
record "not measured" for meshes that are placed 243 and 213 times respectively in
the level. An audit whose refusal rate rises with how simple the mesh is will keep
returning "no answer" precisely on greybox and modular content — the content
reviews are most often run against.

## Root cause guess

Not read from source; the message names a "bounded coarse-grid reference limit
(256)" and a "dense fallback bucket", which reads like a spatial-hash cell
occupancy cap where a triangle spanning many cells is inserted into each, so a
single wall-sized quad alone exceeds the cap. If so the cap is being applied to
*references* (triangle-cell insertions) rather than to *candidate pairs*, and a
grid sized from the model extent would remove the failure for large flat quads.

## Workaround used

None available. I reported the five meshes as unmeasured in the review rather than
as clean, and fell back to `render.detect_z_fighting` from a single camera plus
visual inspection of 12 captures for the level-scale question.

## Distinctness (dedup)

Searched the board for `coarse-grid` / `coarse grid` (no hits) and `z_fighting`
(three hits, none matching):

- `F-render-detect-z-fighting` — the *level-camera* probe verb, a different
  handler and a different question (screen-space pixel differencing from a pose);
  this is the per-asset geometric detector inside
  `geometry.audit_static_meshes`.
- `F-ortho-capture-depth-slab` — orthographic capture depth range; unrelated.
- `B-compile-material-not-tick-gated` — mentions z_fighting only in passing.
- `B-mesh-audit-package-path-reads-as-broken-asset` — same verb, but that is a
  path-resolution defect on the asset *identifier*; this is a check that resolves
  the asset fine and then declines to measure it.

## History
- `#1-filed` `OPEN` reporter — Filed from an ENV blind-A/B critic pass over
  `/Game/FPS/Maps/FPS_Compound`. `geometry.audit_static_meshes {folder:
  "/Game/FPS/Env/Meshes"}` returned `z_fighting` `flagged:2 clean:11 unrunnable:5`
  with the verbatim message *"Large triangle 28 inspected more than the bounded
  coarse-grid reference limit (256); the detector refused to scan a dense fallback
  bucket or report a partial clean result."* on `SM_ENV_WallPanel`, and the same
  message with triangle 6 / 6 / 8 on `SM_ENV_Container20`, `SM_ENV_Crate`,
  `SM_ENV_RoofPanel`; `SM_ENV_Truck` failed differently ("a fighting pair could not
  be reconstructed for region area union"). `static_mesh.describe` on the same
  assets measured 160 / 562 / 486 triangles — so the refusal is triggered by ONE
  large triangle, not by density, and modular blockout kits (large flat quads) are
  the content most likely to trip it. Net effect: `pass:false` on a sweep with zero
  error-severity findings, 28 % of the kit unmeasurable, no cell size, no projected
  area and no threshold knob reported, so the caller has nothing to act on. Asked
  for: run the check (subdivide, or pairwise-fallback — 160 triangles is 12,720
  comparisons), or make the refusal actionable by echoing the grid cell size, the
  offending triangle's projected area, and a caller-settable reference limit the way
  `level.audit` echoes its `thresholds`. No workaround; the review recorded the five
  meshes as unmeasured.
