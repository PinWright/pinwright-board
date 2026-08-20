---
id: B-set-vertex-color-set-all-misses-appended
title: "set_vertex_color set_all=true paints only overlay elements that already exist, so geometry appended after the first colour write stays unpainted — and it still reports the whole mesh as modified"
status: OPEN
severity: High
category: bug
tags: [geometry, set_vertex_color, set_all, vertex-color, overlay, pwmodel, appended-geometry, false-count, silent-wrong, no-diagnostic]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# `set_all` paints elements, not vertices — and lies about how many

`GeometryOps::SetVertexColor` with `bSetAll` is
`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryOps_Elements.cpp:173-191`:

```cpp
if (ColorOverlay->ElementCount() == 0)
{
    ColorOverlay->CreatePerVertex(1.0f);          // :180 — seed skipped when non-empty
}

if (Params.bSetAll)
{
    for (int32 ElementID : ColorOverlay->ElementIndicesItr())   // :186
    {
        ColorOverlay->SetElement(ElementID, Color);
    }
    OutVerticesModified = EditMesh.VertexCount();               // :190
}
```

Two facts combine into the defect:

1. The loop at `:186` walks **overlay elements that already exist**, not mesh vertices. A vertex
   with no `FDynamicMeshColorOverlay` element is unreachable by it.
2. The only call that creates elements, `CreatePerVertex` (`:180`), is guarded on
   `ElementCount() == 0` (`:173`). Once anything populates the overlay — for example a generator's
   `color=`, which the compiler runs with `bSetAll = true` at
   `Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:1039-1047` — the overlay is non-empty
   forever, `CreatePerVertex` never runs again, and vertices appended **after** that point never get
   elements. The guard's comment (`:171-172`) states the intent as "don't clobber a seamed overlay";
   the side effect is that the layer is also never *grown*.

So `extrude_along_spline`, `sweep`, `bevel` and `shell` append triangles from source meshes carrying
no colour overlay, and those triangles render at the overlay default — white — no matter how many
`set_vertex_color set_all` statements follow. Boolean is "unreliable" rather than "never" because a
tool mesh sometimes does carry an overlay (a nested generator with `color=` runs the same `bSetAll`
path at `PwModelCompiler.cpp:1039`), and the merge then contributes elements.

## The count is fabricated

`:190` sets `OutVerticesModified = EditMesh.VertexCount()` — the **entire** mesh, unconditionally,
regardless of how many elements the loop actually touched. That number reaches the wire as
`verticesModified` at
`Source/PinWrightGeometry/Private/Handlers/Geometry/MeshInfoHandler.cpp:433`. The single-vertex
branch three lines below goes out of its way to report a truthful `0` in the analogous case
(`:203-205`); `set_all` does not.

The `.pwmodel` compiler discards the count outright — `PwModelCompiler.cpp:1718-1719` declares
`int32 VerticesModified = 0;`, passes it, and never reads it. No `PWMODEL_*` code exists for
uncoloured geometry (`Source/PinWrightGeometry/Private/Model/PwModelDiagnostic.h` has no such
constant), and no `health` field reports it. A document with a white handrail compiles clean.

## Why the existing test cannot see it

`Source/PinWrightGeometry/Private/Tests/Geometry/TestGeometrySetVertexColorPersistsToOverlay.cpp:16-24`
covers only the fresh-mesh case — `create_box` → `set_vertex_color setAll` → `hasColors == true`.
It never appends geometry after the first write, so the defect sits outside its counterfactual.

## Already documented, not already fixed

`Docs/wiki-src/model.vertex-color.md:56-70` carries the same reach table and the same "renders
white" probe, including shipped consequences: `spiral_stair`'s handrail left white, `oil_lamp`'s
collar `bevel` dropped. The documented workaround is to put `color=` on every generator (`:70`).

**Fix:** grow the overlay instead of seeding it once — on `bSetAll`, create elements for any vertex
that lacks one, then paint. The UV side already has the analogue the colour side is missing:
`ReconcileUVChannelsForAppend` (`PwModelCompiler.cpp:1058`) repairs UV channels after an append, and
there is no colour equivalent. Independently, make the count honest (report elements actually
written, as the single-vertex branch does) and add a `PWMODEL_*` warning when a merged part contains
vertices with no colour element while any colour statement was authored — the count and the warning
are what turn this from invisible into a one-line compile message.

## Not the other vertex-colour ticket

`B-set-vertex-color-wrong-channel-not-persisted` (IN-REVIEW) is a different, now-closed defect: the
verb wrote the legacy `FDynamicMesh3` per-vertex buffer instead of the attribute overlay. The
current code writes the overlay (`GeometryOps_Elements.cpp:151-159`), which is precisely why this
defect is reachable at all.

## History
- `#1-set-all-paints-only-existing-elements` `OPEN` reporter — `GeometryOps::SetVertexColor` with `bSetAll` iterates `ColorOverlay->ElementIndicesItr()` (`GeometryOps_Elements.cpp:186`) — existing overlay *elements*, not mesh vertices — and the one call that creates elements, `CreatePerVertex` (`:180`), is guarded on `ElementCount() == 0` (`:173`). Once any earlier write populates the overlay (a generator's `color=` runs `bSetAll` at `PwModelCompiler.cpp:1039-1047`) the layer is never widened again, so vertices appended afterwards by `extrude_along_spline`, `sweep`, `bevel`, `shell` or a boolean merge carry no element and render at the overlay default — white — however many `set_all` statements follow. Measured: an `extrude_along_spline` rod with `set_all` after it renders white; `bevel` chamfers and `shell` inner walls are missed the same way; boolean is unreliable rather than never, because a tool mesh with its own `color=` contributes elements. The reported count is fabricated: `:190` sets `OutVerticesModified = EditMesh.VertexCount()` — the whole mesh — regardless of what the loop touched, and that reaches the wire as `verticesModified` (`MeshInfoHandler.cpp:433`), while the single-vertex branch three lines below reports a truthful `0` in the analogous case (`:203-205`). Nothing else reports it either: the `.pwmodel` compiler declares and discards the count (`PwModelCompiler.cpp:1718-1719`), no `PWMODEL_*` diagnostic exists for uncoloured geometry, and no `health` field covers it. The only regression test covers a fresh box (`TestGeometrySetVertexColorPersistsToOverlay.cpp:16-24`) and cannot reach the case. Documented but unfixed at `Docs/wiki-src/model.vertex-color.md:56-70`, with `spiral_stair`'s white handrail and `oil_lamp`'s dropped collar bevel as the shipped consequences. Fix: grow the overlay on `bSetAll` rather than seeding once — the UV analogue already exists as `ReconcileUVChannelsForAppend` (`PwModelCompiler.cpp:1058`) and has no colour counterpart — plus an honest count and a `PWMODEL_*` warning for a part carrying colourless vertices. Distinct from `B-set-vertex-color-wrong-channel-not-persisted` (IN-REVIEW), whose legacy-buffer defect is closed at `GeometryOps_Elements.cpp:151-159`.
