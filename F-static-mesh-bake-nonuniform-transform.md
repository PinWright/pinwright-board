---
id: F-static-mesh-bake-nonuniform-transform
title: "No route to a non-uniform mesh bake: static_mesh.bake_transform takes a uniform scalar only, and nothing in the static_mesh namespace can set LOD BuildSettings.BuildScale3D — a per-axis, pivot-preserving bake has to be rebuilt out of geometry.* plus python.execute"
status: OPEN
severity: Low
category: feature
tags: [static-mesh, bake-transform, build-settings, build-scale-3d, non-uniform-scale, pivot, collision, geometry-script]
encounters: 1
lastSeen: 2026-09-09T10:00:00Z
---

# Per-axis scale is the one shape the bake verb refuses, and the fallback is a hand-built pipeline

`static_mesh.bake_transform` (shipped for `F-static-mesh-bake-transform`, IN-REVIEW)
declares `scale` as a scalar and rejects anything else:

```
X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Asset\StaticMeshBakeTransformHandler.cpp:40-63
    RPC_PARAM_OPT("scale", "number", "Uniform scale factor > 0 (default 1)")
    ...
    "scale must be a positive uniform factor; got %g (mirror/zero scale is unsupported)"
```

That scope was a deliberate choice at the time ("no current need" — that ticket's
`#1`/`#2`). There is a need now: making a drone guard **taller without making it wider**
is a per-axis bake, and it has to land in the asset, because a root component cannot be
scaled non-uniformly relative to itself and every instance would otherwise carry the
scale.

The second half of the gap is that no verb reaches `FMeshBuildSettings::BuildScale3D`
either — `BuildScale3D` appears **nowhere** in the plugin source. `geometry.set_lod_settings`
covers reduction knobs (`trianglePercent`, `recomputeNormals`, `recomputeTangents`), not
build settings. `BuildScale3D` is the engine's own answer to exactly this question (it
scales the built render data per axis and is honoured by collision generation), so its
absence removes the cheap route as well as the exact one.

## What the workaround cost

The tall-guard bake was assembled out of `geometry.*` plus `python.execute`:
`CopyMeshFromStaticMesh` -> `TransformMesh` -> `CopyMeshToStaticMesh` for the render mesh,
and `GeometryScript_Collision.transform_simple_collision_shapes` for the simple collision,
with the pivot handled by composing translations around the transform by hand. That is the
same wall `F-static-mesh-bake-transform` `#1` documented for convex hulls, reached from
the other direction: the verb exists now, it just cannot express this transform.

## Ask

Extend `static_mesh.bake_transform` rather than adding a second verb:

- `scale` accepts a per-axis object `{x, y, z}` as well as the current scalar; positive
  components only (mirror stays rejected — winding/normal handling is a separate problem
  and there is still no use for it).
- optional `pivot: {x, y, z}` — the transform is applied about that point, so the bake can
  preserve the authored origin instead of forcing a translation to be computed by the
  caller.
- both applied to render LODs **and** simple collision, exactly as the uniform path
  already does (`FStaticMeshOperations::ApplyTransform` per LOD, hi-res source, AggGeom,
  sockets, bounds).
- optional `overwrite: false` -> write a **new** asset instead of mutating in place, so a
  variant (tall guard) can be derived from a shipped mesh without a manual duplicate first.

Non-uniform scale makes two things load-bearing that the uniform path could ignore, and
whoever implements this should say what it does about each in the response: normals and
tangents need the inverse-transpose, not the transform; and sphere / capsule collision
primitives have no non-uniform form at all, so they must either be converted or reported
as skipped (the handler already has a skipped-and-counted convention for level-set elems).

A separate, smaller ask that would cover many of the same cases: expose
`FMeshBuildSettings.BuildScale3D` through a `static_mesh` (or `geometry`) setter, since
that is a build-time per-axis scale the engine already applies for both render data and
generated collision.

## History
- `#1-tall-guard-needed-per-axis-bake` `OPEN` reporter — Filed on UE 5.8 / `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`, while baking a taller (not wider) sumo guard mesh. `static_mesh.bake_transform` rejects a per-axis scale by construction (`StaticMeshBakeTransformHandler.cpp:46,58-63`), and `BuildScale3D` is absent from the whole plugin source, so neither the exact route nor the engine's build-time route is reachable; the bake was rebuilt from `geometry.*` plus `python.execute` (`GeometryScript_Collision.transform_simple_collision_shapes` for the simple collision), with the pivot composed by hand. Filed as an extension of `F-static-mesh-bake-transform` (IN-REVIEW) rather than appended to it, because that ticket's shipped scope deliberately excludes non-uniform scale and appending would blur what its pending verification covers. `Low`: a real workaround exists and this is a rare authoring path, not an every-session verb.
