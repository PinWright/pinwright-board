---
id: F-geometry-mesh-material-assign
title: "No verb to assign a material to a modeling-tools mesh or its baked Static Mesh — create_box/create_plane/convert_to_static_mesh take no material param and no actor/static_mesh material setter exists"
status: OPEN
severity: Medium
category: feature
tags: [geometry, material, create_box, create_plane, convert_to_static_mesh, static-mesh, actor, material-assignment]
encounters: 1
lastSeen: 2026-07-01T08:50:27.6929557+03:00
---

# The in-editor modeling → bake flow has no material touchpoint anywhere

A user blocking out geometry with the modeling verbs and baking it to a reusable
Static Mesh almost always wants the mesh to carry a real material — that is the
whole point of "make a reusable piece that uses this material". The `geometry`
create/convert flow exposes **no place to name a material at any stage**:

- `geometry.create_box` / `geometry.create_plane` take **no material param** — the
  spawned DynamicMeshActor always comes up on `WorldGridMaterial`.
- `geometry.convert_to_static_mesh` takes **no material param** — the baked
  `/Game` Static Mesh inherits the default material, so the saved asset a user
  drags into levels is un-materialed.
- the `actor` namespace exposes **zero material setters** — there is no
  `actor.set_material` / mesh-component material override verb to bind one after
  the fact.

So the "share one tiling masonry material across both pieces" half of a
blockout-kit intent is not reachable through the MCP: the UVs can be laid for a
single tiling material (`geometry.transform_uvs` works), but nothing can actually
put that material on the mesh or on the baked asset. Both meshes bake with
`WorldGridMaterial` and the UV tiling is never visible on the real material.

## What it should do

Pick one of (any one closes the gap):
1. Add an optional `material` (asset path) param to `geometry.create_box` /
   `geometry.create_plane` (and the sibling primitive creators) that sets the
   spawned DynamicMeshComponent's override material.
2. Add an optional `material` (or `materials: []` per-slot) param to
   `geometry.convert_to_static_mesh` so the baked `/Game` Static Mesh is written
   with its material slots populated.
3. Add a standalone `actor.set_material` / `static_mesh.set_material` verb that
   binds a material asset to a component slot / a StaticMesh slot by index.

Option 2 or 3 is the higher-leverage fix (it makes the **saved asset** correct,
which is what the user drags into levels).

## Evidence (this task — geometry.transform_uvs "brick kit" build, 14 calls, outcome clean)

Story: build a brick wall panel + matching floor tile that **share one tiling
masonry material**, lay their UVs, and bake each to `/Game/GeneratedMeshes/SM_*`.
The UV + bake half ran clean (create_box → transform_uvs → convert_to_static_mesh
→ static_mesh.describe, all `ok:true`). The friction was the material step:

> "there is NO MCP verb to bind the shared masonry material to a dynamic mesh
> actor or its baked static mesh — create_box/create_plane take no material param,
> convert_to_static_mesh takes no material param, and the actor namespace exposes
> zero material setters — so both meshes baked with the default WorldGridMaterial.
> The UVs are laid for a single shared tiling material, but the 'share one masonry
> material' step was not achievable through the MCP as a real user without an
> escape hatch (property.set)."

The agent did not force it (material binding is not in the readback success
check), so both `SM_BrickPanel` and `SM_FloorTile` were baked un-materialed. The
per-finding judge filed the unrelated readback bug `B-mesh-info-empty-nonfinite-json`;
this capability gap was not filed.

**Workaround:** `property.set` on the StaticMesh `StaticMaterials` array or the
DynamicMeshComponent override materials — a source-dive-level struct write, not a
first-class verb; the agent judged it out of scope and skipped it.

severity rationale: impact=soft-blocker (doable only via a source-dive property.set workaround) × reach=common blockout/bake intent but not every session -> Medium

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.transform_uvs` "brick kit" task (14 calls, outcome clean, judge filed the unrelated `B-mesh-info-empty-nonfinite-json`). PROCESS/capability finding: the create → transform_uvs → convert_to_static_mesh flow has no material touchpoint at any stage — `geometry.create_box`/`create_plane` take no material param, `geometry.convert_to_static_mesh` takes no material param, and `actor` exposes no material setter — so both baked `/Game/GeneratedMeshes/SM_*` assets came up on `WorldGridMaterial` and the "share one tiling masonry material" intent was unreachable without a `property.set` escape hatch the agent skipped. Proposed: add an optional `material`/`materials[]` param to the geometry create verbs and/or `convert_to_static_mesh`, or a standalone `actor.set_material`/`static_mesh.set_material` verb (option 2/3 preferred — fixes the saved asset). Dedup: distinct from `B-mesh-info-empty-nonfinite-json` (mesh-info readback bug), from `E-create-procedural-terrain-no-material-echo` (terrain material *echo*, not assignment), from `B-material-stub-handlers-silent-success` / `B-material-authoring-save-no-disk-write` (material-graph authoring, not mesh binding), and from `F-geometry-uv-prep-pipeline-batch` / `F-staticmesh-texture-dsl-sidecars` (UV-op batch / dump sidecars); no existing mesh-material-assignment ticket on the board.
