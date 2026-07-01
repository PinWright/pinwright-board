---
id: B-geometry-convert-static-mesh-drops-collision
title: "geometry.convert_to_static_mesh silently drops the DynamicMeshActor's simple collision during the bake — generate_collision shapeCount:1 becomes zero collision elements on the StaticMesh"
status: IN-REVIEW
severity: Medium
category: bug
tags: [geometry, convert_to_static_mesh, generate_collision, static-mesh, collision, silent-data-loss, bake]
encounters: 1
lastSeen: 2026-06-30T23:47:57.2366488+03:00
---

# `geometry.convert_to_static_mesh` discards the generated simple collision during the bake — the baked StaticMesh has zero collision elements despite a successful `generate_collision`

"Give it collision, then bake it into a StaticMesh I can drop into a level" is
the canonical tail of a procedural-prop authoring chain. On this path the
collision the caller explicitly generated is silently lost in the bake:

1. `geometry.generate_collision {actorName:"RecessedPanel", collisionType:"box"}`
   → `{collisionType:"box", shapeCount:1, message:"Collision generated"}` —
   a box body lands on the **DynamicMeshActor**.
2. `geometry.convert_to_static_mesh {assetPath:"/Game/SciFiKit/SM_RecessedWallPanel"}`
   → `{exists:true, created:true, class:"StaticMesh", triangleCount:108, ...}` —
   full success, no collision-related field, no warning.
3. `static_mesh.describe {assetPath:"/Game/SciFiKit/SM_RecessedWallPanel"}`
   → `collision:{ elements:{ sphere:0, box:0, sphyl:0, convex:0, taperedCapsule:0 } }`,
   `collisionTraceFlag: CTF_UseDefault` — **zero** simple collision on the baked
   asset.

The box that `generate_collision` reported (`shapeCount:1`) never reaches the
baked StaticMesh. Both action calls report success, so nothing in-band signals
the loss — only a follow-up `static_mesh.describe` (whose `collision.elements`
field exists thanks to `B-static-mesh-missing-collision-body-counts`, DONE)
reveals box:0. A caller who asked for a collidable, droppable prop gets a
collision-less asset and trusts it.

## Likely cause

`geometry.convert_to_static_mesh` bakes via
`UGeometryScriptLibrary_CreateNewAssetFunctions::CreateNewStaticMeshAssetFromMesh`
(see `MeshOpsHandler.cpp`, the convert handler), which builds a fresh
`UStaticMesh` from the DynamicMesh geometry only. The source DynamicMeshActor's
`UBodySetup::AggGeom` (where `generate_collision` wrote the box) is not copied
into the new StaticMesh's `BodySetup`, so the simple collision is dropped. The
bake carries geometry but not the generated collision primitives.

## What it should do

- Carry the DynamicMeshActor's simple collision (`AggGeom` box/sphere/sphyl/
  convex) into the baked StaticMesh's `BodySetup` so a prior `generate_collision`
  survives the bake — or expose an explicit `bakeCollision` flag (that would be
  an F capability) if transferring by default is undesirable.
- At minimum, surface the discard in-band: when the source actor has simple
  collision that is not carried over, the `convert_to_static_mesh` response
  should report it (e.g. a `collisionElements`/`collisionDropped` field or a
  warning) instead of an unqualified `created:true`, so the caller knows a
  separate collision pass on the StaticMesh is required.
- Docs floor (if the drop is intended): note on the
  `convert_to_static_mesh` overlay in `docs/wiki-src/geometry.md` that
  `generate_collision` runs on the DynamicMeshActor and does **not** transfer
  through the bake — call `generate_collision` semantics / set collision on the
  StaticMesh after converting.

## Repro (verbatim from the seed task, replayable)

```
geometry.create_box {name:"RecessedPanel", width:400, height:300, depth:20}
... inset / extrude / bevel / recalculate_normals (geometry only) ...
geometry.generate_collision {actorName:"RecessedPanel", collisionType:"box"}
  -> {collisionType:"box", shapeCount:1, message:"Collision generated"}
geometry.convert_to_static_mesh {actorName:"RecessedPanel",
    assetPath:"/Game/SciFiKit/SM_RecessedWallPanel"}
  -> {exists:true, created:true, class:"StaticMesh", triangleCount:108, vertexCount:56}
static_mesh.describe {assetPath:"/Game/SciFiKit/SM_RecessedWallPanel"}
  -> collision.elements all 0 (box:0), collisionTraceFlag:CTF_UseDefault
```

**Workaround:** do not trust `convert_to_static_mesh`'s success as proof of
collision; after baking, verify with `static_mesh.describe` (`collision.elements`)
and add collision to the StaticMesh separately, since the DynamicMeshActor's
`generate_collision` box does not survive the bake.

severity rationale: impact=silent data loss of the generated collision on the
bake path (convert reports `created:true` with no collision signal), but it is
detectable via `static_mesh.describe`'s `collision.elements` and collision is
re-derivable, so a soft blocker with a workaround = Medium × reach=terminal bake
verb of every procedural-prop chain, common but not every-session -> no change
-> Medium.

## Dedup

Distinct from `B-geometry-convert-static-mesh-no-disk-write` (IN-REVIEW,
Critical) — that ticket is about the **whole `.uasset` never being flushed to
disk** (the asset vanishes on cold load); here the asset persists and is
readable, but its **collision** is absent. Different root cause (no `BodySetup`
copy, vs. no `UPackage::Save`). Distinct from
`B-static-mesh-missing-collision-body-counts` (DONE) — that fix **added** the
`collision.elements` readback field to `static_mesh.json`/`describe` (which is
precisely how this drop is now observable, box:0); it did not touch the
geometry bake path. Not the `inset`/`outset` direction bug
(`B-geometry-inset-outset-direction-swapped`).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the geometry.inset seed
  task (build a recessed sci-fi wall panel, give it collision, bake to
  StaticMesh). `geometry.generate_collision {collisionType:"box"}` reported
  `shapeCount:1` on the DynamicMeshActor and `geometry.convert_to_static_mesh`
  reported `created:true`/`exists:true`, yet `static_mesh.describe` on the baked
  `/Game/SciFiKit/SM_RecessedWallPanel` showed `collision.elements` all 0
  (box:0, `CTF_UseDefault`) — the simple box collision did not transfer into the
  bake. Both action calls report success with no in-band signal of the loss; the
  count-only success check (tris/verts strictly increase) passes regardless. The
  attempt's own friction note: "generate_collision reported shapeCount:1 box on
  the DynamicMeshActor yet the baked StaticMesh's static_mesh.describe shows 0
  simple collision elements ... the simple box collision did not transfer into
  the bake." Likely cause: the bake via
  `CreateNewStaticMeshAssetFromMesh` rebuilds geometry only and does not copy the
  source actor's `BodySetup`/`AggGeom`. Fix: carry the simple collision through
  the bake (or add a `bakeCollision` flag), and at minimum report the discard in
  the convert response. Trace was otherwise efficient (12 first-try action/verify
  calls, no retries/python). Surfaced alongside the inset-direction bug
  (`B-geometry-inset-outset-direction-swapped`) in the same task.
- `#2-collision-transfer` `IN-REVIEW` developer — Root cause confirmed in source:
  `geometry.convert_to_static_mesh` bakes via `CreateNewStaticMeshAssetFromMesh(Target.Mesh, ...)`
  (`MeshOpsHandler.cpp` convert handler), which rebuilds geometry only and leaves the baked
  StaticMesh's `BodySetup` empty; `generate_collision` writes the box onto the source
  DynamicMeshComponent's BodySetup (`GenerateAndApplyCollision` → `SetSimpleCollisionOfDynamicMeshComponent`),
  which the bake never reads — so the box was dropped (`static_mesh.describe` box:0). Fix:
  added `GeometryUtils::TransferSimpleCollisionToStaticMesh(const UBodySetup*/UDynamicMeshComponent*,
  UStaticMesh*)` in `Handlers/Geometry/CollisionHelpers.h` (copies the source BodySetup's
  `AggGeom` + `CollisionTraceFlag` onto the baked mesh's own BodySetup, `InvalidatePhysicsData` +
  `MarkPackageDirty`, returns the carried element count). Wired it into the convert handler
  BEFORE the existing `SaveAssetToDiskReportingPresence` save so the collision lands in the
  `.uasset`, and echoed the count as a new `collisionElements` field on the success response
  (in-band signal so `created:true` no longer implies collision that was not transferred).
  Files: `Source/PinWright/Private/Handlers/Geometry/CollisionHelpers.h`,
  `Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp`. Regression test:
  `Source/PinWright/Private/Tests/Geometry/TestGeometryConvertStaticMeshCarriesCollision.cpp`
  (`PinWright.geometry.convert_to_static_mesh.CarriesCollision`) drives create_box →
  generate_collision box → convert through the real dispatcher, asserts the response reports
  `collisionElements >= 1`, and loads the baked StaticMesh to assert its `BodySetup->AggGeom`
  actually holds the box (`BoxElems.Num() >= 1`) — reverting the transfer leaves both zero.
</content>
