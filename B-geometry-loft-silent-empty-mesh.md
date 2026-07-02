---
id: B-geometry-loft-silent-empty-mesh
title: "geometry.loft and geometry.sweep report success but emit an empty mesh (null-polygon sweep stub)"
status: IN-REVIEW
severity: High
category: bug
tags: [silent-empty-mesh-success]
encounters: 1
lastSeen: 2026-07-02T05:22:47.0291434+03:00
---

# geometry.loft and geometry.sweep report success but emit an empty mesh (null-polygon sweep stub)

Two advertised modeling verbs in `AdvancedMeshOpsHandler.cpp` share a copy-pasted
"build a cross-section polygon, then sweep it" stub that silently produces **no
geometry** while still reporting a clean success:

- `geometry.loft` reports success with `profilesUsed` equal to the number of
  profile actors passed, but the target DynamicMesh is left completely empty
  (`trianglesBefore:0, trianglesAfter:0`; `get_mesh_info` → `vertexCount:0,
  triangleCount:0`).
- `geometry.sweep` reports success with a plausible `sweepStatus` string
  (e.g. `"Linear sweep with 16 steps, height 100.0"`) and `pathSteps`/
  `profileVertices` counts, but likewise never changes the mesh
  (`trianglesAfter == trianglesBefore`).

In both cases the caller trusts the success signal and moves on to save/convert a
mesh that has no geometry at all. There is no error, no warning, no diagnostic — a
silent false-success on the verbs' only geometry-producing paths.

## Root cause (shared across three code sites)

Each site builds a correct `TArray<FVector2D> PolygonVertices`, then round-trips it
through a **default-constructed** `FGeometryScriptSimplePolygon` before (in one
case) handing it to `AppendSweepPolygon`:

```cpp
FGeometryScriptSimplePolygon Polygon;
for (const FVector2D& V : PolygonVertices)
{
    if (Polygon.Vertices.IsValid()) { Polygon.Vertices->Add(V); }   // guard is always false
}
```

`FGeometryScriptSimplePolygon::Vertices` is a `TSharedPtr<TArray<FVector2D>>` that
is **null until `Reset()` allocates it** (engine struct at
`GeometryScriptTypes.h:588-613`). The handler never calls `Polygon.Reset()`, so
`Polygon.Vertices.IsValid()` is `false` on the fresh struct and every guarded `Add`
(and the guarded copy-out to `PolygonVerts2D`) is skipped — the polygon stays empty.

The three affected sites (all in `AdvancedMeshOpsHandler.cpp`):

1. **`geometry.loft` multi-profile branch** (was lines 346-388): copies the empty
   polygon into `PolygonVerts2D` and passes *that* to `AppendSweepPolygon` → the
   sweep appends nothing. `profilesUsed` is still set unconditionally.
2. **`geometry.loft` no-profile branch** (was lines 427-438): builds the same dead
   polygon **and never calls `AppendSweepPolygon` at all** — it only `UE_LOG`s the
   frame count, so the bounding-box extrusion produces nothing.
3. **`geometry.sweep`** (was lines 594-609): the same copy-paste — builds the dead
   polygon plus real path frames (spline or linear fallback), then **only `UE_LOG`s
   and never calls `AppendSweepPolygon`**, so the whole verb is a silent no-op
   regardless of the null-polygon bug.

The intermediate `FGeometryScriptSimplePolygon` serves no purpose: the correct
pattern already lives in the same file — the spline `extrude_along_spline` handler
(was ~lines 1003/1018) passes its `PolygonVertices` array **directly** as the 4th
argument of `AppendSweepPolygon` (no round-trip) and produces real geometry.

## Out of scope (split to a feature ticket)

Even with a valid polygon, the multi-profile loft is **not a true loft**: it uses
only the **first and last** profile actors, approximates the cross-section as a
*circle* of the first profile's max XY extent (`ProfileRadius =
FMath::Max(ProfileExtent.X, ProfileExtent.Y)`), and sweeps that single circle in a
straight line between the two actor locations. All intermediate profiles and every
profile's actual silhouette are ignored. Making the verb *produce geometry* fixes
the empty-mesh bug reported here; making it honor each profile's cross-section is a
separate enhancement tracked as **F-geometry-loft-true-multiprofile**.

severity rationale: impact=silent-false-success (caller trusts success + `profilesUsed`/`sweepStatus` and saves an empty asset) x reach=every invocation of two advertised modeling verbs hits these paths -> High.

**Workaround:** Use `geometry.revolve` (sweep a silhouette polyline around an axis)
to build vase/turned/swept shapes; it produces a real mesh. `geometry.loft` and
`geometry.sweep` currently cannot produce any geometry.

**Fix:** In `AdvancedMeshOpsHandler.cpp`, drop the default-constructed
`FGeometryScriptSimplePolygon` round-trip at all three sites (its `Vertices`
TSharedPtr is null, silently dropping every vertex) and pass `PolygonVertices`
straight to `AppendSweepPolygon` — matching the existing spline-sweep handler. Wire
`AppendSweepPolygon` into the loft no-profile branch and into `geometry.sweep`,
which previously built a polygon and path frames but only logged. (Implementing a
true multi-profile loft is deferred to F-geometry-loft-true-multiprofile.)

## History
- `#1-initial-repro` `OPEN` reporter — geometry.loft silently emits an empty mesh. Replay: create_procedural_mesh name=VaseLoft; create_disc Prof_Foot(z=0,r=16,seg=24) / Prof_Belly(z=45,r=52) / Prof_Neck(z=95,r=24) / Prof_Rim(z=130,r=34); then loft {actorName:"VaseLoft", profileActors:["Prof_Foot","Prof_Belly","Prof_Neck","Prof_Rim"], subdivisions:8, smooth:true, cap:true} -> success `{"profilesUsed":4,"trianglesBefore":0,"trianglesAfter":0}`; get_mesh_info VaseLoft -> `{"vertexCount":0,"triangleCount":0}`. Root cause confirmed in source: default-constructed FGeometryScriptSimplePolygon has a null Vertices TSharedPtr (GeometryScriptTypes.h:588-593), so the guarded Add/copy at AdvancedMeshOpsHandler.cpp:346-365 are all no-ops and AppendSweepPolygon receives an empty polygon.
- `#2-reword-and-fix-sweep-stub-family` `IN-REVIEW` developer — Reworded from loft-only to the shared null-polygon sweep-stub family after adversarial review found the same stub in the loft no-profile branch and in geometry.sweep (both build a dead polygon and never call AppendSweepPolygon), and that the ticket bundled a separate "true multi-profile loft" feature. Fix in `Plugins/PinWright/Source/PinWright/Private/Handlers/Geometry/AdvancedMeshOpsHandler.cpp`: removed the default-constructed FGeometryScriptSimplePolygon round-trip at all three sites and pass PolygonVertices straight into AppendSweepPolygon (the pattern the spline extrude_along_spline handler already uses); added the missing AppendSweepPolygon call to the loft no-profile branch and to geometry.sweep. Descoped the real multi-profile loft to new feature ticket F-geometry-loft-true-multiprofile. Regression test `PinWright.geometry.LoftSweepEmitGeometryNotSilentEmpty` (`Tests/Geometry/TestGeometryLoftSweepEmitGeometry.cpp`) drives all three paths through the real dispatcher against in-code DynamicMeshActor fixtures (two sphere profiles at distinct Z for the loft; empty procedural mesh for the sweep linear fallback; a box for the no-profile extrusion) and asserts each emits geometry (trianglesAfter > 0, or > trianglesBefore for the append-to-box case) — it fails if any of the three fixes is reverted.
