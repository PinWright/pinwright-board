---
id: F-geometry-mesh-material-assign
title: "No verb assigns a material to a Static Mesh asset slot — geometry.convert_to_static_mesh bakes geometry only and no static_mesh material setter exists, so the saved asset comes up on the default material"
status: IN-REVIEW
severity: Medium
category: feature
tags: [geometry, material, convert_to_static_mesh, static-mesh, static_mesh, material-assignment, bake]
encounters: 1
lastSeen: 2026-07-01T08:50:27.6929557+03:00
---

# No verb assigns a material to a baked Static Mesh asset slot

A user blocking out geometry with the modeling verbs and baking it to a reusable
Static Mesh almost always wants the **saved asset** to carry a real material — that
is the whole point of "make a reusable piece that uses this material". The bake
path drops the material entirely:

- `geometry.convert_to_static_mesh` bakes the DynamicMesh **geometry only**
  (`CreateNewStaticMeshAssetFromMesh`, MeshOpsHandler.cpp), so the `/Game` Static
  Mesh it saves always comes up with its slot on the **default material** — the
  asset a user drags into levels is un-materialed.
- The `static_mesh` namespace exposed **only** `static_mesh.describe` — no verb
  bound a material to a StaticMesh's `StaticMaterials` slot, so once baked there was
  no first-class way to give the saved asset a material.

So the "share one tiling masonry material across both pieces" half of a
blockout-kit intent was not reachable for the **saved asset**: the UVs can be laid
for a single tiling material (`geometry.transform_uvs` works), but nothing wrote
that material into the baked StaticMesh's slots.

## What is NOT the gap (corrected from the initial report)

The initial report also claimed the **live** dynamic-mesh actor had no material
setter and that the only escape hatch was "a source-dive-level struct write". That
is **inaccurate**: `actor.set_component_properties {actorName,
componentName:"DynamicMeshComponent", properties:{OverrideMaterials:["/Game/…"]}}`
is a documented first-class verb that binds a material to the live
`UDynamicMeshComponent` (`ApplyJsonValueToProperty` loads each `OverrideMaterials`
element by asset path, then `MarkRenderStateDirty()`). The live-actor bind is
reachable today. What it does **not** do is carry into the baked StaticMesh —
`convert_to_static_mesh` bakes geometry only, so an `OverrideMaterials` set on the
component before baking is not written into the baked asset's slots. The genuine,
un-workaroundable gap is therefore the **saved StaticMesh material slot**, not the
live actor.

## What it should do

Add a `static_mesh.set_material` verb that binds a material asset to a StaticMesh
`StaticMaterials` slot by index and persists the asset — mirroring the established
`<namespace>.set_material` convention (`spline.set_spline_mesh_material`,
`landscape.set_material`, `water.set_water_body_material`). This closes the
modeling→bake loop: `geometry.convert_to_static_mesh` → `static_mesh.set_material`.

**Workaround (before this fix):** `actor.set_component_properties` on the LIVE
DynamicMeshComponent's `OverrideMaterials` binds the material for viewport display,
but does **not** reach the baked/saved StaticMesh; `property.set` on the baked
mesh's `StaticMaterials` (a `TArray<FStaticMaterial>` struct array) is an awkward
struct write with no clean first-class path.

**Fix:** Added `static_mesh.set_material`
(`Source/PinWright/Private/Handlers/Asset/StaticMeshSetMaterialHandler.cpp`): loads
the StaticMesh + material, validates the slot index (errors
`INVALID_MATERIAL_INDEX` on out-of-range rather than silently no-op'ing, since
`UStaticMesh::SetMaterial` no-ops on an invalid index), calls
`UStaticMesh::SetMaterial(index, material)`, force-saves via
`SaveAssetToDiskReportingPresence`, and echoes the bound slot (read off the asset)
plus an honest `AddAssetSaveReport` verdict.

## Evidence (initial report — geometry.transform_uvs "brick kit" build, 14 calls, outcome clean)

Story: build a brick wall panel + matching floor tile that **share one tiling
masonry material**, lay their UVs, and bake each to `/Game/GeneratedMeshes/SM_*`.
The UV + bake half ran clean (create_box → transform_uvs → convert_to_static_mesh
→ static_mesh.describe, all `ok:true`). The friction was the material step: both
`SM_BrickPanel` and `SM_FloorTile` baked with the default material because there
was no verb to bind the shared masonry material to the **baked** StaticMesh, and
`convert_to_static_mesh` takes no material param. (The initial report additionally
asserted the live actor had no material setter — see the correction above; the
real, remaining gap is the saved-asset slot.)

The agent did not force it (material binding is not in the readback success check),
so both baked meshes were left un-materialed. The per-finding judge filed the
unrelated readback bug `B-mesh-info-empty-nonfinite-json`; this capability gap was
not filed at the time.

severity rationale: impact=soft-blocker (the modeling→bake pipeline could never put
a material on its OUTPUT asset; only an awkward `property.set` on a struct array
reached it) × reach=common blockout/bake intent but not every session -> Medium

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.transform_uvs` "brick kit" task (14 calls, outcome clean, judge filed the unrelated `B-mesh-info-empty-nonfinite-json`). PROCESS/capability finding: the create → transform_uvs → convert_to_static_mesh flow has no material touchpoint at any stage — `geometry.create_box`/`create_plane` take no material param, `geometry.convert_to_static_mesh` takes no material param, and `actor` exposes no material setter — so both baked `/Game/GeneratedMeshes/SM_*` assets came up on `WorldGridMaterial` and the "share one tiling masonry material" intent was unreachable without a `property.set` escape hatch the agent skipped. Proposed: add an optional `material`/`materials[]` param to the geometry create verbs and/or `convert_to_static_mesh`, or a standalone `actor.set_material`/`static_mesh.set_material` verb (option 2/3 preferred — fixes the saved asset). Dedup: distinct from `B-mesh-info-empty-nonfinite-json` (mesh-info readback bug), from `E-create-procedural-terrain-no-material-echo` (terrain material *echo*, not assignment), from `B-material-stub-handlers-silent-success` / `B-material-authoring-save-no-disk-write` (material-graph authoring, not mesh binding), and from `F-geometry-uv-prep-pipeline-batch` / `F-staticmesh-texture-dsl-sidecars` (UV-op batch / dump sidecars); no existing mesh-material-assignment ticket on the board.
- `#2-static-mesh-set-material` `IN-REVIEW` developer — Reworded (adversarial lens): the initial "actor namespace exposes zero material setters, only a source-dive `property.set` works" premise is false — `actor.set_component_properties {componentName:"DynamicMeshComponent", properties:{OverrideMaterials:[path]}}` already binds a material to the live `UDynamicMeshComponent` as a first-class verb (ComponentHandler → `ApplyJsonValueToProperty` load-by-path → `MarkRenderStateDirty`). Narrowed the ticket to the genuine, un-workaroundable gap: the SAVED StaticMesh material slot (convert bakes geometry only, so the baked asset stays on the default material and nothing wrote its `StaticMaterials`). Implemented a new `static_mesh.set_material` verb (`Source/PinWright/Private/Handlers/Asset/StaticMeshSetMaterialHandler.cpp`): loads the StaticMesh + `UMaterialInterface`, validates the slot index and errors `INVALID_MATERIAL_INDEX` on out-of-range (`UStaticMesh::SetMaterial` silently no-ops otherwise → would be a fake success), calls `UStaticMesh::SetMaterial(index, material)`, force-saves via `SaveAssetToDiskReportingPresence`, and echoes the bound slot (read off the asset) + `AddAssetSaveReport` honest-persistence verdict — mirrors the `spline.set_spline_mesh_material` / `landscape.set_material` precedent and closes the `convert_to_static_mesh` → `set_material` loop. Regression test `PinWright.static_mesh.set_material.AssignsSlot` (`Source/PinWright/Private/Tests/Assets/TestStaticMeshSetMaterialAssignsSlot.cpp`) builds every fixture in-code (create_box → convert_to_static_mesh bakes the StaticMesh; `material.authoring.create_material` makes the material), assigns slot 0, and asserts the reloaded `UStaticMesh` slot-0 `MaterialInterface` is the assigned material (fails if the verb / `SetMaterial` call is reverted), plus an out-of-range-index → `INVALID_MATERIAL_INDEX` case.
