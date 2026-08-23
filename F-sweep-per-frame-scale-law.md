---
id: F-sweep-per-frame-scale-law
title: "sweep / extrude_along_spline interpolate the cross-section LINEARLY between scale_start and scale_end, so no radius law that is not a straight line can follow a path — the engine's per-frame FTransform scale slot was always there, only the caller's two scalars were not"
status: IN-REVIEW
severity: Medium
category: feature
tags: [pwmodel, geometry, sweep, extrude_along_spline, scales, taper, scale-curve, cross-section]
encounters: 1
lastSeen: 2026-08-23T00:00:00Z
---

# A swept tube could taper only straight

`sweep` and `extrude_along_spline` published exactly two scale controls, `scale_start` and
`scale_end`, and both ops turned them into one line per frame:

```cpp
const float Scale = FMath::Lerp((float)Params.ScaleStart, (float)Params.ScaleEnd, Alpha);
PathFrames.Add(FTransform(Rotation, Location, FVector(Scale)));
```

So the cross-section could only ramp **linearly** from one end of the path to the other. Every
radius law that is not a straight line was unreachable along a path:

- a **power-law taper**, `r(t) = r_tip + (r_base - r_tip) * (1 - t)^p`, which is what a trunk,
  a mast, a spire or a tapered column actually is;
- an **exponential flare** near one end;
- a **waist** or a **bulge** — non-monotonic, so no pair of endpoint scalars can produce it at
  all.

The documented way round it was to split the shape into one sweep per span and match the radii at
each join by hand. That costs more geometry, buries a cap inside every joint, and is one
arithmetic slip away from a visible ledge at each one — the same banding that got a cone-chain
mesh rejected elsewhere in this project's history.

**The engine never required any of it.** Both ops already build one `FTransform` per path frame,
and that transform's scale slot has always been per-frame. Only the caller's two scalars were not.

## What landed

`scales=[(alpha, scale), …]` on both ops, as a `PointList2` — a value kind the format already
had, so no new kind and no grammar change.

- `alpha` is the **normalised position** along the path: 0 at the first frame, 1 at the last.
  Not a distance in uu.
- Piecewise-linear between knots; **held, not extrapolated,** outside the outermost pair.

Holding is load-bearing rather than a convenience. Extrapolating a curve that stops at
`alpha 0.9` reaches a **negative** scale at 1.0, which reflects the cross-section and reverses
the swept tube's facing normals — while `isClosed`, `boundaryEdges` and the triangle count all
stay identical. That is a mesh that renders correctly and lights inside-out, i.e. exactly the
class of defect `health.signedVolume` exists to catch after the fact. Holding cannot produce it:
every value it can return is a value the author wrote.

Refusals, each because the repair would be a guess about which half the author meant, all
reported as line-anchored `PWMODEL_OP_FAILED` / `ERR_INVALID_PARAMS`:

| refused | why not repaired |
|---|---|
| `scales` beside `scale_start` / `scale_end` | only one can be honoured; accepting the pair leaves the scalars looking set while the curve silently wins — the shape `PWMODEL_MATERIAL_ID_CONFLICT` already refuses on `append_buffers` |
| fewer than 2 knots | one knot is a constant scale, which omitting the parameter already gives |
| `alpha` outside `[0, 1]` | alpha is normalised, so 1.5 is not a longer path — it is a knot the sweep never reaches, and clamping would move it |
| `alpha` not strictly ascending | one frame per alpha means one scale there; two knots at one alpha describe a step the sweep cannot make, and a descending pair is nearly always a transposed line |
| `scale <= 0` | 0 collapses the section onto the path; negative reflects it and reverses facing normals invisibly |

`[(0, a), (1, b)]` is **exactly** `scale_start=a scale_end=b` — asserted vertex for vertex in the
tests — so `scales` is a superset rather than a second dialect.

It does not resample `path=` (alpha is evaluated at the frames the author wrote), and it is
dropped along with `cap`, `scale_start` and `scale_end` on an `extrude_along_spline` path that
returns to its start, because the engine's loop branch gates path scaling off entirely. That drop
now **warns on its own account**: the existing `cap` warning mentions the scale parameters but
fires only when `cap` was also set, so an author who wrote a radius law on a ring and no cap got
a uniform tube with nothing said — the same failure mode this op already went through once, on
`cap` itself.

No RPC spelling, the same position `profile=` is in: neither `geometry.sweep` nor
`geometry.extrude_along_spline` can carry a list of knots, so both RPC paths are unchanged.

## Implementation

Plugin commit `a2587117`:

- `FSweepScaleCurve` (knots + `IsSet()` + `Evaluate()`) on `GeometryOps_Advanced.h`, a field on
  both `FSweepParams` and `FExtrudeAlongSplineParams`.
- One shared `GeometryOpsAdvanced_ScaleAt()` used by all **three** alpha-to-scale sites — the
  spline branch, the vertical fallback, and `ExtrudeAlongSpline` — so the curve cannot end up
  honoured in two of three places, which is how `scale_start` / `scale_end` came to be dropped on
  a closed path without anything saying so.
- `ReadSweepScaleCurve()` in `PwModelCompiler.cpp` carrying the five refusals, wired into both
  ops beside `ReadSweepProfile`.
- Op-table entries on both ops in `PwModelParser.cpp`, so `model.describe_ops` publishes it.
- `Docs/pwmodel-format.md` and `Docs/wiki-src/model.authoring.md`.
- Six automation tests, `Tests/Model/TestPwModelSweepScaleCurve.cpp`: the law in isolation
  (knots exact, both spans' slopes distinct, held past both ends, unset curve is the identity),
  the geometry it produces measured at every frame of a waisted sweep, the scalar equivalence,
  the op-table publication on both ops, a document round trip, and nine refusals.

**NOT LINKED and the suite has NOT been run.** Compile-checked with `-SingleFile` on all four
touched translation units (all `Result: Succeeded`); a link build plus a suite run are pending,
which is why this is `IN-REVIEW` and not `DONE`.

## History
- `#1-linear-only-taper-along-a-path` `OPEN` reporter — hit while building a tapering form from
  ops against a documented radius law. `revolve profile=` takes an arbitrary radius law but has
  no path (its axis is the local Z), and `sweep` takes an arbitrary path but only a straight
  ramp; nothing in the vocabulary carries a law ALONG a path. Worked around in the source by
  splitting the run into two radius-matched sweeps, which holds the law to under 1 uu and cannot
  ledge only because the join's two radii are equal by construction — an invariant the author has
  to maintain by hand at every joint.
- `#2-scales-knot-list-implemented` `IN-REVIEW` developer — added `scales=[(alpha, scale), …]` to
  both ops as a `PointList2`, piecewise-linear and held outside its outermost knots; five
  refusals (conflict with the scalars, <2 knots, alpha outside `[0, 1]`, alpha not strictly
  ascending, non-positive scale); one shared alpha-to-scale helper across all three call sites;
  a new warning when an `extrude_along_spline` loop drops the law, which the existing `cap`
  warning covered only when `cap` was also set. Defaults and every existing document are
  unchanged — the two-knot curve reproduces `scale_start` / `scale_end` vertex for vertex, which
  is asserted rather than assumed. Plugin commit `a2587117`. Compile-checked with `-SingleFile`
  on all four files; **not linked, suite not run** — needs a tester to run the six new tests
  after the next link build.
