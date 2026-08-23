---
id: B-set-vertex-color-set-all-misses-appended
title: "set_vertex_color set_all=true paints only overlay elements that already exist, so geometry appended after the first colour write stays unpainted — and it still reports the whole mesh as modified"
status: IN-REVIEW
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
- `#2-additional-live-ue58-overlay-gap` `OPEN` reporter — Additional evidence: **Adversarial review A — current UE 5.8 source confirms the defect, but the report needs scope correction.** Actuality: CONFIRMED CURRENT. Framing: `GeometryOps::SetVertexColor` still seeds only when `ElementCount()==0`, paints existing overlay elements, and reports `EditMesh.VertexCount()` (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\GeometryOps_Elements.cpp:167-191`); the RPC exposes that count (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\MeshInfoHandler.cpp:421-438`). The compiler colors generator scratch meshes, then reconciles only UVs before `AppendMesh` (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Model\PwModelCompiler.cpp:981-1074`, `:825-860`). UE 5.8 leaves appended triangles unset when the source lacks the target attribute (`C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Private\DynamicMesh\DynamicMeshAttributeSet.cpp:226-251`, `C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Public\DynamicMesh\DynamicMeshOverlay.h:284-289`), so later colorless corners remain white and a following `set_all` cannot reach them. Current docs still record white extrude/bevel/boolean regions with no diagnostic or health (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Docs\wiki-src\model.vertex-color.md:95-115`). However, `oil_lamp` reports its shell wall reached by `set_all` (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Examples\pwmodel\oil_lamp.pwmodel:452-462`), so “shell always misses” is too broad; scope should be operation/order-dependent sparse-overlay gaps. Proposed fix: INCOMPLETE, because growing missing elements at `bSetAll` does not repair later compiler appends, a naïve `CreatePerVertex` clears seams (`C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Private\DynamicMesh\DynamicMeshOverlay.cpp:88-90`, `:167-169`), and the compiler still needs append policy, honest counts, diagnostics, and regression coverage. Evidence: fresh/full and single-vertex tests do not cover sparse `set_all` append behavior (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Tests\Geometry\TestGeometryOpsElements.cpp:219-359`); overlay persistence remains fresh-box only (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Tests\Geometry\TestGeometrySetVertexColorPersistsToOverlay.cpp:16-25,94-138`); no missing-color diagnostic is declared (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Model\PwModelDiagnostic.h:122-155`). Runtime: NOT VERIFIED. Recommendation: REFRAME; keep OPEN/High, correct current citations and scope, then design append-safe seam-preserving color reconciliation, truthful `verticesModified`, a model diagnostic, focused tests, and a later UE 5.8 VertexColorViewMode probe.
- `#3-additional-adversarial-b` `OPEN` reporter — Additional evidence: **Adversarial review B — A's core verdict survives; its append-path citation and fix boundary need correction.** Actuality: CONFIRMED CURRENT. Framing: I agree the current op paints only existing primary-color overlay elements and publishes the whole mesh vertex count (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\GeometryOps_Elements.cpp:167-191`; `X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\MeshInfoHandler.cpp:421-438`). “Shell always misses” is false: the oil-lamp shell begins with an empty overlay and its later `set_all` reaches the wall (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Examples\pwmodel\oil_lamp.pwmodel:452-462`); the defect is order-dependent sparse overlay coverage. I disagree only with treating `DynamicMeshAttributeSet.cpp:226-251` as the compiler's direct append path: generator appends use `FDynamicMeshEditor::AppendMesh`; `FDynamicMesh3::AppendTriangle` initializes new color-triangle storage unset (`C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Private\DynamicMesh\DynamicMesh3_Edits.cpp:189-215`, `C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Private\DynamicMesh\DynamicMeshAttributeSet.cpp:1243-1258`), and colors copy only when the source overlay exists (`C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Private\DynamicMeshEditor.cpp:2016-2025`, `:2268-2292`). Proposed fix: INCOMPLETE, not a band-aid; a `bSetAll` growth patch fixes explicit recolor after append but not later generators, `append_buffers` color/no-color mixing, compiler scratch-to-target reconciliation, honest affected counts, or diagnostics/tests. Runtime: NOT VERIFIED. Recommendation: REFRAME; keep OPEN/High, define whether unset corners are painted with a seam-preserving default, then implement reconciliation at every append/modifier/merge boundary and add sparse-corner tests.

- `#4-overlay-grown-and-count-made-honest` `IN-REVIEW` developer — Chose **implement**. `GeometryOps::SetVertexColor` under `bSetAll` now GROWS the colour overlay before painting: it walks triangles, and any triangle with no element assignment gets one per corner, then every element is painted (GeometryOps_Elements.cpp). Review A's seam objection is handled by construction rather than by accepting the risk — `CreatePerVertex` clears elements, so it is not used; only UNASSIGNED triangles are touched, a corner whose vertex already carries exactly one element REUSES it (no new seam), and an already-split vertex gets a fresh corner because there is no basis for guessing which side a new triangle belongs to. Nothing rebuilds an existing assignment. The fabricated count is fixed too: `OutVerticesModified` was `EditMesh.VertexCount()` unconditionally and is now the distinct parent vertices of the elements actually written, so a vertex no triangle references is not claimed as painted. Added `colorElementsCreated` to the response (and an optional out-param on the op, leaving the .pwmodel compiler's call site unchanged) — non-zero exactly when the mesh had grown since the last colour write, which is the in-band signal the ticket says is missing. Review A/B are correct that this does not cover every append path — the compiler's scratch-to-target reconciliation, `append_buffers` colour/no-colour mixing, and a `PWMODEL_*` diagnostic are NOT done and remain open work; what is fixed is that an explicit `set_all` after an append now reaches the appended geometry, which is the ticket's title claim. Test: Tests/Geometry/TestGeometrySetVertexColorReachesAppendedGeometry.cpp, id `.set_vertex_color.SetAllReachesGeometryAppendedAfterTheFirstWrite` — paint, append_triangle, paint again, reading the overlay off the mesh rather than the response, with a mid-sequence known-bad control asserting the appended triangle genuinely arrives uncoloured so the final assertion cannot pass for the wrong reason. Docs/wiki-src/geometry.md gains a `### geometry.set_vertex_color` section. Docs/wiki-src/model.vertex-color.md's reach table is NOT yet corrected — it records measurements against the old behaviour and will be re-measured rather than edited from inference. Commits 2efa8206, 4a902563, 4f7110e3. Compile-checked -SingleFile, Result: Succeeded. Vision verification pending a link window.
