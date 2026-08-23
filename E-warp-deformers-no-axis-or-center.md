---
id: E-warp-deformers-no-axis-or-center
title: "bend / twist / taper hardcode the warp frame to FTransform::Identity, so all three only work on geometry that is Z-aligned AND sits on the part-local origin — the engine call takes a full FTransform and harmonic_deform, the sibling deformer, already publishes axis= and center="
status: OPEN
severity: Medium
category: enhancement
tags: [pwmodel, geometry, bend, twist, taper, warp-deformer, axis, center, harmonic_deform, parameter-gap]
encounters: 1
lastSeen: 2026-08-23T00:00:00Z
---

# The three warp deformers can only warp about world Z through the origin

`GeometryOps::Bend`, `Twist` and `Taper`
(`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryOps_Modeling.cpp`, the "Warp
deformers" block) each end in the same shape:

```cpp
UGeometryScriptLibrary_MeshDeformFunctions::ApplyBendWarpToMesh(
    Mesh, BendOptions, FTransform::Identity, Params.Angle, Params.Extent, nullptr);
```

with the comment *"The warp frame is the mesh's own origin; the deformer's extent is measured
about it."* `ApplyTwistWarpToMesh` and `ApplyFlareWarpToMesh` are called the same way. That third
argument is the engine's **gizmo frame** — position *and* orientation — and it is the only thing
that decides which axis the deform runs along and where its extents are measured from. The engine
ops themselves say so: `TwistMeshOp.cpp` opens `// Twists along the Z axis`, `BendMeshOp.cpp`
opens `// Bends along the Y-axis` and then maps gizmo-Z into it, and `FMeshSpaceDeformerOp`
carries a `GizmoFrame` / `SetTransform` pair for exactly this purpose.

Pinning it to identity means the three ops are usable **only** on geometry that is aligned to the
part-local Z axis and straddles (or starts at) the part-local origin. Nothing in the format says
so, and nothing warns.

## What it costs an author

- A limb, rail, beam, pipe run or handrail authored horizontally, or at an `at=` away from the
  origin, cannot be bent, twisted or tapered along its own axis at all. `taper` in particular is
  the natural way to give any swept or revolved form a flare, and it is unreachable for every
  such form.
- The workaround exists and is undocumented: author the shape at the origin along +Z, warp it,
  then `transform at= rotate=` it into place. That is discoverable only by someone who already
  knows the constraint, and it does not help at all when the shape must be warped *after* being
  combined with something else in the same part — which is exactly when an author reaches for a
  warp deformer rather than baking the shape.
- `symmetric_extents=false` + `lower_extent` was added to answer "a mesh that does not straddle
  the origin", but it only re-splits the span along the **same** axis. It cannot answer "the mesh
  is not on that axis at all", which is the more common case.

## The precedent is already in the tree

`harmonic_deform` — the other deformer in the same vocabulary — publishes both `axis=` (an enum,
`x`/`y`/`z`) and `center=` (a point on the axis, part-local), and its documentation argues the
case: *"center= is a point on the axis in part-local space... this axis is the one the geometry
was revolved about, and a box-derived centre would move whenever an earlier op changed the mesh's
extent."* Every word of that applies to the three warps, which currently have neither field.

## Fix

Publish `axis=` (enum, default `z` — the current behaviour) and `center=` (vector3, default
`(0, 0, 0)` — the current behaviour) on `bend`, `twist` and `taper`, and build the FTransform from
them instead of passing identity:

```cpp
const FTransform WarpFrame(FRotationMatrix::MakeFromZ(AxisVector).ToQuat(), Params.Center);
```

Both defaults reproduce today's behaviour exactly, so no existing document changes. Spell the
enum and the vector the same way `harmonic_deform` spells them, so the vocabulary has one
spelling for "which axis" and one for "through which point" rather than two.

Worth stating in the docs at the same time: the extent is measured **along that axis from that
centre**, which is the sentence that makes `symmetric_extents` / `lower_extent` legible for the
first time.

## Related

- `E-geometry-warp-extent-semantics` — the extent semantics of the same three ops. Same family,
  different field; that one is about what `extent` measures, this one about what it measures
  *along*.

## History
- `#1-warp-frame-pinned-to-identity` `OPEN` reporter — found while building a tapering, twisting
  form from ops. `harmonic_deform` + `twist` composed correctly for a Z-aligned, origin-centred
  base (that composition is the documented way to get an azimuthal period that varies along the
  axis), but every off-axis or off-origin piece of the same model was unreachable for all three
  warps, because `GeometryOps_Modeling.cpp` passes `FTransform::Identity` as the gizmo frame in
  all three wrappers. Engine side confirms the frame is the whole control: `TwistMeshOp.cpp`
  `// Twists along the Z axis`, `BendMeshOp.cpp` `// Bends along the Y-axis` with an explicit
  gizmo-Z-to-Y-up matrix, `FMeshSpaceDeformerOp::GizmoFrame` / `SetTransform`. Proposed fix is
  `axis=` + `center=` with defaults that reproduce today's behaviour byte for byte, spelled
  exactly as `harmonic_deform` already spells them.
