---
id: E-warp-deformers-no-axis-or-center
title: "bend / twist / taper hardcode the warp frame to FTransform::Identity, so all three only work on geometry that is Z-aligned AND sits on the part-local origin — the engine call takes a full FTransform and harmonic_deform, the sibling deformer, already publishes axis= and center="
status: IN-REVIEW
severity: Medium
category: enhancement
tags: [pwmodel, geometry, bend, twist, taper, warp-deformer, axis, center, harmonic_deform, parameter-gap]
encounters: 2
lastSeen: 2026-08-27T18:57:03+05:00
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

## Encounter 2026-08-27 — the prescribed workaround has an unstated precondition, and it fails silently

Nothing above changes; this section is added beside the existing text. Evidence measured
2026-08-27 on UE 5.8 in this checkout, while building `SM_Column_Broken_A`.

The workaround this ticket prescribes at § *What it costs an author* is:

> author the shape at the origin along +Z, warp it, then `transform at= rotate=` it into place

**`transform rotate=` on geometry that is no longer at the origin is itself a trap.** The
`transform` op's rotation **pivots about the part-local origin**, so it does not turn the geometry
in place — it swings it around the origin on a lever arm of `distance * sin(angle)`. Verified in
source at HEAD: `PwModelCompiler.cpp:2401-2406` calls `TransformMesh(Mesh, ReadTransformParams(P),
…)`, and `ReadTransformParams` (`PwValueRead.h:276-281`) returns `FTransform(rotate, at, scale)`,
which applies scale, then rotation, then translation — all about the origin. Measured instance: a
tool authored at `z=1790`, tilted 7 degrees, was displaced `1790 * sin(7deg) = 218 uu` in X, cut the
wrong part of the model, and compiled `success: true` with the entire documented health gate green.

**Write the precondition onto the workaround:** it is safe only while **the shape is still at the
part-local origin when the `transform` runs** — i.e. the `transform` must carry the shape's *first*
placement, composing `rotate=` and `at=` in one op. It fails the moment the shape already has an
`at=` (or any other translation) before it, which is the common case for a limb, rail or beam
authored in place, and exactly the case this ticket's readers are trying to solve.

If `axis=` / `center=` land on the three warps as proposed, the workaround becomes unnecessary
rather than merely conditional, which is a further argument for the fix.

Filed separately as `E-pwmodel-transform-rotate-pivots-origin` — same premise as this ticket (a
`.pwmodel` op silently taking the part-local origin as its frame, undocumented and unwarned),
different op and the opposite failure: these three warps fail to REACH off-origin geometry, while
`transform rotate=` reaches it and DISPLACES it.

## Related

- `E-geometry-warp-extent-semantics` — the extent semantics of the same three ops. Same family,
  different field; that one is about what `extent` measures, this one about what it measures
  *along*.
- `E-pwmodel-transform-rotate-pivots-origin` — the op this ticket's workaround routes through.
  See the encounter section above.

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
- `#2-workaround-precondition-correction` `OPEN` reporter — Additional evidence and a correction
  to this ticket's prescribed WORKAROUND; no status change, existing text untouched, detail in
  the `Encounter 2026-08-27` section above. Measured 2026-08-27 on UE 5.8 in this checkout while
  building `SM_Column_Broken_A`. The workaround "author the shape at the origin along +Z, warp
  it, then `transform at= rotate=` it into place" routes through an op that carries the SAME
  origin-as-frame premise this ticket is about: `transform rotate=` pivots about the part-local
  origin, so on geometry that is no longer at the origin it swings the shape on a lever arm of
  `distance * sin(angle)` instead of turning it in place. Source-verified at HEAD:
  `PwModelCompiler.cpp:2401-2406` calls `TransformMesh(Mesh, ReadTransformParams(P), …)` and
  `ReadTransformParams` (`PwValueRead.h:276-281`) returns `FTransform(rotate, at, scale)`, which
  applies scale, then rotation, then translation, all about the origin. Measured: a boolean tool
  at `z=1790` tilted 7 degrees was displaced `1790*sin(7deg) = 218` uu in X, cut the wrong part
  of a radius-155 shaft, and compiled `success: true` with `isClosed: true`, positive
  `signedVolume`, `boundaryEdges: 0`, `degenerateTriangles: 0`, `nonManifoldVertices: 0` — the
  whole documented health gate — while leaving 168 triangles hanging 140 uu clear; only
  `floatingGeometry` reported it. The unstated precondition, now written onto the workaround: it
  is safe ONLY while the shape is still at the part-local origin when the `transform` runs, i.e.
  the `transform` carries the shape's first placement. It fails as soon as the shape has an `at=`
  before it — the common case for a limb, rail or beam authored in place, which is this ticket's
  own motivating case. Filed separately as `E-pwmodel-transform-rotate-pivots-origin` (same
  premise, different op, opposite failure: these warps fail to REACH off-origin geometry,
  `transform rotate=` reaches it and DISPLACES it). `encounters` 1 -> 2, `lastSeen` refreshed.
- `#3-frame-already-landed-extent-docs-corrected` `IN-REVIEW` developer — "The proposed fix is
  ALREADY IN THE TREE at HEAD and this entry does not re-implement it; it verifies it and closes
  the one clause left open. Verified present: `GeometryOps::FWarpFrameSpec { EMeshAxis Axis = Z;
  FVector Center = ZeroVector; }` on `FBendParams` / `FTwistParams` / `FTaperParams`
  (`GeometryOps_Modeling.h`), `GeometryOpsModeling_WarpFrame()` building the gizmo frame from a
  cyclic basis (frame X = (Axis+1)%3, frame Y = (Axis+2)%3, translation = Center) and all three
  ops passing it instead of `FTransform::Identity` (`GeometryOps_Modeling.cpp`), `axis=` (enum via
  the shared `AxisValues()`, default `z`) and `center=` (vector3, default `(0, 0, 0)`) published
  on all three ops spelled exactly as `harmonic_deform` spells them (`PwModelParser.cpp`), read
  through `ReadWarpFrame` (`PwModelCompiler.cpp`) and, on the RPC side, `ReadWarpFrameSpec` +
  `PW_WARP_FRAME_RPC_PARAMS` (`MeshOpsHandler.cpp`). Landed in `9a89ddaf`; the regression suite
  `TestGeometryDeformerFrameAndScale.cpp` landed in `4199860f` and already carries the
  byte-identical-defaults assertion this ticket demands —
  `PinWright.Geometry.Ops.WarpFrame.DefaultFrameReproducesTheHardcodedIdentity` runs each of the
  three ops at default params against the raw engine call at `FTransform::Identity` and asserts a
  max positional delta of exactly `0.0`, plus
  `...WarpFrame.AxisChoosesWhichAxisTheExtentSpans` and
  `...WarpFrame.CenterTranslatesTheDeformWithTheMesh`. Wiki `Docs/wiki-src/geometry.md` documents
  both parameters. WHAT WAS ACTUALLY CHANGED HERE: only the ticket's remaining docs clause — six
  `extent` parameter descriptions still read `about the origin`, which stopped being true the
  moment `center=` shipped. Rewritten to `measured ALONG axis FROM center` in the three
  `.pwmodel` op specs (`PwModelParser.cpp` bend / twist / taper) and to `measured along \`axis\`
  from \`center\`` in the three RPC schemas (`MeshOpsHandler.cpp` geometry.bend / .twist /
  .taper). String literals only, no behaviour change; the `symmetric half-extent` /
  `[-extent, +extent]` / `[-lowerExtent, +extent]` markers the
  `infra.wiki_handler.*.Geometry*ExtentSemantics` doc tests grep for are all preserved. NOT
  COMPILED and NOT RUN — a build and a `PinWright.Geometry.*` + `PinWright.Model.*` +
  `PinWright.infra.wiki_handler.*` pass is the verification this entry is asking for."
