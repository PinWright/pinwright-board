---
id: E-pwmodel-bend-displacement-direction-unstated
title: "bend / twist / taper publish axis= for the axis the extent SPANS, but nothing says which direction the deformer PUSHES — it falls out of the cyclic basis (axis=z displaces along +Y) and there is no parameter to choose it, so bending a vertical form fore-and-aft needs an author-side authoring-axis swap plus a transform"
status: OPEN
severity: Medium
category: enhancement
tags: [pwmodel, geometry, bend, twist, taper, warp-deformer, axis, displacement-direction, docs, parameter-gap]
encounters: 1
lastSeen: 2026-09-02T23:45:00+03:00
---

# `axis=` names the axis being bent, not the direction the bend goes — and the direction is unreachable

`E-warp-deformers-no-axis-or-center` asked for `axis=` / `center=` on the three warp deformers and
they shipped (that ticket's `#3` entry verifies them in the tree). This is the layer underneath:
the parameter that shipped answers **which axis the extent is measured along**, and an author
reaching for `bend` needs a second answer the op does not publish — **which way the geometry
moves**.

The op table says, in full:

```
axis    The axis the extent is measured along. Defaults to z, which is what this op always used.
        Spelled the same as harmonic_deform's, because a deformer family whose members name
        their axis differently is one an author has to learn twice.
```

That is accurate and it is not the question. A bend has two axes — the one being bent along and
the one being bent *towards* — and only the first is named anywhere in the op table, the wiki, or
the ticket that added it. The existing regression test is even called
`PinWright.Geometry.Ops.WarpFrame.AxisChoosesWhichAxisTheExtentSpans`, which states the covered
half precisely and leaves the other half untested and unsaid.

## Measured

Two `model.validate` probes, no asset written, on this project's UE 5.8 editor:

```
part magazine {
    box size=(4.6, 3.2, 18.6) segments=(2, 2, 16) at=(11.4, 0, -5.7)
    bend angle=18 axis=z center=(11.4, 0, -5.7) extent=9.3 bidirectional=true
}
```

`bounds.size` came back `x 4.599999`, `y 3.905887`, `z 19.024195` against an input of
`(4.6, 3.2, 18.6)`. **X is untouched to six decimals and Y widened by 0.706.** `axis=z` bends
towards **+Y**, and `success: true`, `isClosed: true`, `signedVolume: 273.77`,
`selfIntersections: 0`, zero diagnostics — nothing in the response mentions a direction at all.

Swapping the cross-section to `size=(3.2, 4.6, 18.6)` and adding
`transform rotate=(0, 0, -90) at=(11.1, 0, -5.2)` produced the intended fore-and-aft curve, at
`z size 20.00`, closed, 0 self-intersections.

## Why it is the cyclic basis, not an accident

`E-warp-deformers-no-axis-or-center` `#3` records the implementation:
`GeometryOpsModeling_WarpFrame()` builds the gizmo frame from **a cyclic basis — frame
X = (Axis+1)%3, frame Y = (Axis+2)%3, translation = Center**. With `axis=z` (index 2) that makes
frame Y = world Y, and `BendMeshOp.cpp` opens `// Bends along the Y-axis`. So the displacement
direction is fully determined, deterministic and reproducible — it is simply never written down,
and the cyclic basis means it also **rotates with `axis=`**: `axis=x` will push along Z and
`axis=y` along X, so an author who learns "bend goes +Y" from a `axis=z` probe learns something
that is false for the other two.

## What it costs an author

A form that must bend along a *named* direction has to be authored on whichever axis the cyclic
basis happens to pair with it and then rotated into place. On the model that surfaced this — an
AR-pattern magazine, vertical, curving fore-and-aft — that meant authoring the fore-aft dimension
on local Y, bending, then `transform rotate=(0, 0, -90)`, and carrying a comment explaining why
the box's `size` reads transposed. That workaround also collides with
`E-pwmodel-transform-rotate-pivots-origin`: the `transform` is only safe while the shape is still
at the part-local origin, so the generator cannot carry its own `at=` and the placement has to
move into the `transform` too.

None of that is discoverable. The failure mode when it is guessed wrong is the quiet one this
format keeps producing: the mesh bends the wrong way, stays closed and manifold with positive
signed volume and zero self-intersections, and only a render shows it.

## What it should do

Either of these closes it; the first is docs-only.

1. **State the direction in the `axis` description**, per axis, and in `Docs/wiki-src/geometry.md`
   — e.g. "the deform displaces along the NEXT axis in the cyclic order X→Y→Z: `axis=x` pushes
   along Z, `axis=y` along X, `axis=z` along Y." Cheap, and it is the fact an author needs at the
   moment they type the op.
2. **Publish a `direction=` (or `towards=`) enum** on `bend`, defaulting to the current cyclic
   partner so nothing existing changes. The engine already takes a full `FTransform` gizmo frame
   and `GeometryOpsModeling_WarpFrame()` already builds one, so this is a choice of basis vector
   rather than new machinery — the same shape as the `axis=` addition that ticket `#3` landed.

A regression test named for the uncovered half — `...WarpFrame.AxisChoosesWhichWayTheDeformPushes`
— beside the existing `AxisChoosesWhichAxisTheExtentSpans` would pin it.

## History

- `#1-filed-from-ar-magazine` `OPEN` reporter — Found while authoring
  `Content/FPS/Weapons/Meshes/SM_WPN_AR.pwmodel` for the FPS weapons stream. The magazine was
  first built as five stacked boxes on a 4-degree-per-segment arc; that spelling passes every
  health field and renders as a visible staircase (corner ledges up to 0.99 uu on a body 3.2
  across), so it was replaced with a single `bend`ed box. Getting the bend to run fore-and-aft
  cost two probes and a read of `E-warp-deformers-no-axis-or-center` `#3` to work out that the
  direction comes from a cyclic basis. Filed as a separate ticket rather than as History on that
  one because that ticket's ask — publish `axis=` and `center=` — is implemented and verified;
  this is a distinct parameter gap sitting on top of it, and reopening a landed ticket would
  misreport the state of the work that shipped.
