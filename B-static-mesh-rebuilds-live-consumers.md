---
id: B-static-mesh-rebuilds-live-consumers
title: "Three main-module StaticMesh mutators rebuild render data without quiescing live Niagara mesh consumers"
status: OPEN
severity: High
category: bug
tags: [static-mesh, nanite, lod, bake-transform, niagara, render-state, raytracing, crash]
---

# StaticMesh rebuilds leave non-StaticMeshComponent proxies on freed render data

## What's wrong

The main module already documents the hazard in `Utils/MeshRenderConsumerScan.h`: Niagara scene
proxies cache a StaticMesh's render data and the engine does not reregister them during an in-place
mesh rebuild. `model.compile` uses `PinWrightMeshRebuild::FQuiesceScope` to destroy/flush/recreate
those consumers; `static_mesh.describe` reports the same candidate set.

Three scoped mutators do not use that guard:

- `asset.generate_lods`: `Mesh->Build()` + `PostEditChange()` at
  `AssetWorkflowHandler.cpp:1201-1202`.
- `asset.nanite_rebuild_mesh`: Nanite notification/build + compilation drain at `:1364-1392`.
- `static_mesh.bake_transform`: full `PostEditChange` rebuild + drain at
  `StaticMeshBakeTransformHandler.cpp:252-256`.

Concrete path: a live Niagara mesh renderer holds the target's old `FStaticMeshRenderData`; one of
these verbs rebuilds and frees it; the next ray-tracing gather dereferences the stale proxy. That
path has already produced a render-thread fatal through `model.compile`
(`B-model-compile-live-niagara-mesh-renderer-raytracing-assert`).

## What it should do

Move/share the quiesce half where the main module can use it, bracket the complete rebuild and
save, and refuse with the established not-quiescable error if any live consumer retains render
state. Do not merely add a tick safe point; position and consumer lifetime are separate guards.

## Workaround

Remove or deactivate every live Niagara component before using these mutators.

## Related

- `B-set-lod-settings-unguarded-rebuild` covers Geometry-module LOD setters, not these verbs.

## History
- `#1-pattern-scan` `OPEN` reporter — Source and existing-crash-path confirmation only; no editor, build, test, or RPC run was performed.
