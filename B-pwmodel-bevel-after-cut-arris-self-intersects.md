---
id: B-pwmodel-bevel-after-cut-arris-self-intersects
title: "pwmodel `bevel` on a mesh whose polygroup edges a boolean has already CUT THROUGH produces a self-intersecting surface, and the only signal is a model-wide PWMODEL_SELF_INTERSECTING_SURFACE with line -1 that never names bevel"
status: OPEN
severity: High
category: bug
tags: [pwmodel, model.compile, model.validate, bevel, boolean, self-intersection, silent-wrong-geometry, no-diagnostic, health-gate]
encounters: 1
lastSeen: 2026-09-03T04:45:00+03:00
---

# `bevel` after a cut that crosses an arris self-intersects, and no diagnostic points at the op

`bevel` already warns about the two polygroup shapes it cannot handle — no polygroups
(no-op) and one-group-per-quad (notches every quad boundary). There is a **third** shape it
cannot handle and it warns about none of it: a mesh whose polygroup edges have been
**interrupted by a boolean**. Bevelling one produces a surface that passes through itself.

Every other health field stays clean, so the only signal is the model-wide
`PWMODEL_SELF_INTERSECTING_SURFACE`, which carries `line: -1`, `part: ""` and no mention of
`bevel`. On a multi-part document with several booleans there is nothing to tell the author
which op did it.

## Minimal repro (run live via `model.validate`, which creates nothing)

An 8 x 4.2 x 1 bar with ONE 0.58 slot cut through the top, then bevelled:

```
pwmodel 0

materials { Receiver = "/Game/FPS/Weapons/Materials/MI_WPN_MetalAnodised" }

part one_slot {
    box size=(8.0, 4.2, 1.0) at=(0, 0, 8.5) material="Receiver" color=(0.082, 0.082, 0.087, 1)
    subtract {
        box size=(0.58, 5.0, 0.55) at=(0, 0, 8.85) material="Receiver" color=(0.082, 0.082, 0.087, 1)
    }
    bevel distance=0.12 segments=0 infer_material_id=true
    uv channel=0 mode=box scale=(1, 1)
}
```

```
meshTriangleCount: 96
health: isClosed true, boundaryEdges 0, degenerateTriangles 0, nonManifoldVertices 0,
        orientationConsistent true, signedVolume 32.18 (positive),
        selfIntersections 22, selfIntersectingComponents 1
diagnostics: 1 x PWMODEL_SELF_INTERSECTING_SURFACE, line -1, part "", text does not
             contain the word "bevel"
```

The reported first crossing is `(0.41, -2.1, 8.455)`. Both coordinates are the bevel's own
inset: the slot wall is at x 0.29 and 0.29 + 0.12 = **0.41**; the slot floor is at z 8.575 and
8.575 - 0.12 = **8.455**; y -2.1 is the untouched flank plane. So the crossing is the bevel's
corner patch at the point where the cut interrupts the flank's polygroup boundary.

## Three controls that isolate it to the interrupted edge

| Spelling | Triangles | selfIntersections |
|---|---|---|
| `box 37 x 4.2 x 1` + `bevel 0.12` (no boolean at all) | 44 | **0** |
| `box 8 x 4.2 x 1` + subtract a **blind** pocket (tool 2.0 across a 4.2 bar, reaches no arris) + `bevel 0.12` | 96 | **0** |
| `box 8 x 4.2 x 1` + `union` a boss standing on a face + `bevel 0.12` | 156 | **0** |
| `box 8 x 4.2 x 1` + subtract a slot **through** the top (tool 5.0 across a 4.2 bar) + `bevel 0.12` | 96 | **22** |

It is not "bevel after any boolean" — it is specifically a boolean that cuts an edge the bevel
then has to walk.

## It is topological, not a tuning problem

Measured on the same 37 x 4.2 x 1 bar with 34 slots (`array_linear count=34 offset=(1.083,0,0)`):

- `bevel distance=0.12` -> 10,784 triangles, `selfIntersections` **4096 (truncated)**
- `bevel distance=0.05` -> 10,784 triangles, `selfIntersections` **4096 (truncated)** — byte-identical counts
- `weld_vertices tolerance=0.001` before it -> **identical** 10,784 / 4096
- `weld_vertices tolerance=0.01 only_unique_pairs=false` before it -> identical failure
- `bevel ... segments=1` -> triangle count changes (96 -> 194 on the one-slot case), crossing
  count does **not** (22 -> 22)

So neither distance, nor welding, nor rounding subdivisions is a workaround. The wiki's
`weld_vertices`-before-`bevel` advice (`Examples/pwmodel/spur_gear.pwmodel:225-231`) addresses a
different symptom — a chamfer stopping dead at a seam — and does not touch this one.

## Scale in real documents

Encountered while adding the edge breaks a review asked for on `Content/FPS/Weapons/Meshes/`:

| Part | bevel placed after its booleans |
|---|---|
| `SM_WPN_AR` part `rail` (34 slots) | 4096+ crossing pairs |
| `SM_WPN_AR` part `upper` (4 unions + 1 subtract) | 107 |
| `SM_WPN_AR` part `handguard` (hollow + 9 M-LOK slots) | 25 |
| `SM_WPN_Pistol` part `sights` (a second unfiltered bevel over an already-bevelled sibling) | 215 |

All four came back `success: true` with `isClosed: true`, `boundaryEdges: 0`,
`nonManifoldVertices: 0`, `degenerateTriangles: 0` and positive per-part `signedVolume`. A
health gate written on any field except `selfIntersections` passes every one of them.

## Workaround, and why it is not a fix

Bevel each solid **while it is still a whole primitive** — immediately after its generator, and
inside its own `union { }` block for anything added later — so every boolean lands on an
already-chamfered solid. That is what both weapons now do, and both compile with
`selfIntersections: 0`. It costs geometry where a cut then crosses the chamfer (the 34-slot
rail goes 5,110 -> 13,106 triangles instead of 10,784) and it means cut edges cannot be
chamfered at all.

## What it should do

Either handle the interrupted edge, or — much cheaper and consistent with what `bevel` already
does for its other two bad-input shapes — **warn at the `bevel` line**. The op has the
information: it knows the mesh, and the compiler knows a boolean ran earlier in the same part.
A line-anchored warning naming `bevel` and the earlier boolean would turn a silent broken
surface into an authoring message, the way `PWMODEL_EXTRUDE_FACING_OPPOSED` does for the
inside-out `extrude`.

At minimum, `PWMODEL_SELF_INTERSECTING_SURFACE` should carry the owning part and, when the
crossing witness lands on geometry a `bevel` produced, say so. Today it carries `line: -1` and
`part: ""`.

severity rationale: impact=silent broken surface that passes every other health field and the
documented gate's first two terms x reach=`bevel` after a boolean is the ordinary way to break
edges on any hard-surface model -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found while adding 0.1-0.15 uu edge breaks to
  `SM_WPN_AR` and `SM_WPN_Pistol` under `Content/FPS/Weapons/Meshes/`. Minimal case reduced to
  one box + one through-slot + `bevel` = 22 crossing pairs, against 0 for the same box with no
  boolean, with a blind pocket, or with a unioned boss. Invariant under `distance` (0.05 and
  0.12 give identical counts), under `weld_vertices` at two tolerances and both
  `only_unique_pairs` settings, and under `segments`. Worked around by bevelling every solid
  before its booleans; both weapons now compile `selfIntersections: 0`.
