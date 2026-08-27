---
id: B-model-compile-live-niagara-mesh-renderer-raytracing-assert
title: "model.compile on a static mesh a live Niagara mesh renderer is drawing kills the editor in the ray-tracing gather"
status: OPEN
severity: Critical
category: bug
tags: [model, static-mesh, niagara, mesh-renderer, ray-tracing, editor-crash]
encounters: 1
lastSeen: 2026-08-27
---

# Recompiling a mesh that a spawned Niagara mesh renderer references crashes the editor

`model.compile` rebuilds and re-saves the `UStaticMesh` in place. If a Niagara actor in the open
level is currently rendering that mesh through a `UNiagaraMeshRendererProperties`, the render
thread's ray-tracing instance gather runs against the mesh's now-empty LOD array on the next frame
and hits a hard assert. The whole editor process dies, taking every other agent's unsaved work with
it.

## Symptom

```
Assertion failed: (Index >= 0) & (Index < ArrayNum)
  [File: Engine/Source/Runtime/Core/Public/Containers/Array.h] [Line: 1339]
Array index out of bounds: -1 into an array of size 0
```

Callstack, innermost first (render thread):

```
FNiagaraRenderableStaticMesh::GetRayTraceLODModelData()   NiagaraMeshRendererProperties.cpp:234
FNiagaraRendererMeshes::GetDynamicRayTracingInstances()   NiagaraRendererMeshes.cpp:1676
FNiagaraSystemRenderData::GetDynamicRayTracingInstances() NiagaraSystemRenderData.cpp:253
RayTracing::FDynamicRayTracingInstancesContext::GatherDynamicRayTracingInstances_Internal()
                                                          RayTracing/RayTracing.cpp:826
RayTracing::FinishGatherInstances()                       RayTracing/RayTracing.cpp:2190
FDeferredShadingSceneRenderer::Render()                   DeferredShadingRenderer.cpp:2315
RenderViewFamily_RenderThread()                           SceneRendering.cpp:5232
```

`-1 into an array of size 0` is the LOD index: the mesh has no render data at the instant the gather
runs, so the "highest available LOD" computation returns -1 and indexes an empty `LODResources`.

## Repro

Reproduced once, deterministically in sequence, on `/Game/Maps/Atlantis`:

1. `model.compile {filePath: "Content/Atlantis/Meshes/SM_Bubble.pwmodel"}` — writes
   `/Game/Atlantis/Meshes/SM_Bubble`.
2. Point a Niagara mesh renderer at it and spawn it into the open level:
   `niagara.set_property {assetPath: "/Game/Atlantis/VFX/NS_Bubbles_Ambient",
     target: {kind:"renderer", emitter:"Fountain", index:0},
     propertyPath: "Meshes[0].Mesh", value: "/Game/Atlantis/Meshes/SM_Bubble.SM_Bubble"}`
   then `niagara.spawn_actor` + `effect.activate_niagara` so it is drawing.
3. Edit `SM_Bubble.pwmodel` (here: `subdivisions=3` -> `subdivisions=4`) and call
   `model.compile` on it **again**, with the Niagara actor still live in the viewport.

The compile itself returns success (`meshTriangleCount: 108`, `saved: true`). The process dies on a
subsequent render-thread frame.

**Evidence** — `Saved/Logs/EAContentExamples58.log`, crash at `2026.08.27-14.29.37` UTC
(19:29 local; logs are UTC+0, machine is UTC+5). The `model.compile` that preceded it is the last
PinWright line before the assert.

## Impact

Critical. It kills the editor for every agent sharing it, and everything not yet saved is gone. The
trigger is an ordinary iteration loop — author mesh, look at it in the level, adjust the mesh,
recompile — which is exactly what a particle mesh needs several passes of. Nothing in
`model.compile`'s response or docs warns that a live Niagara mesh renderer makes the call unsafe.

It is narrower than "recompiling a mesh in use": a static-mesh *actor* referencing the same mesh does
not crash (the normal `UStaticMesh` reregister path handles it). It is specifically the Niagara mesh
renderer's ray-tracing gather, which reads LOD data without checking that render data exists.

## Workaround

Before recompiling a mesh any Niagara system renders, remove or stop the consumers:
`actor.delete` the Niagara preview actors (or `effect.deactivate_niagara` and let the particles
expire), recompile, then re-spawn. Disabling ray tracing (`r.RayTracing 0`) would also avoid the
specific gather, but that changes what every other agent's captures look like and is not safe on a
shared editor.

## Suggested fix

The assert is in engine code, so PinWright cannot fix it directly, but it can stop reaching it:

- In `model.compile`, before swapping render data, find `UNiagaraComponent`s in the open world whose
  renderers reference the target `UStaticMesh` and mark them inactive / re-register them around the
  rebuild (`FComponentReregisterContext` equivalent), the way the static-mesh editor's own rebuild
  path does.
- Failing that, refuse with a typed error naming the live Niagara actors, so the caller deletes them
  first instead of losing the editor. A refusal that names the blocking actors is strictly better
  than a crash, and matches the house rule that an error must say what was checked and what to do.

## History
- `#1-initial-repro` `OPEN` reporter — Hit while authoring `/Game/Atlantis/VFX/NS_Bubbles_Ambient`
  on the Atlantis map build. `SM_Bubble` was recompiled from `subdivisions=3` to `subdivisions=4` to
  fix a visibly hexagonal silhouette, while `PWTEST_Bub2` was live in the viewport rendering it
  through a Niagara mesh renderer. Compile reported success; editor died on the next render-thread
  frame in the ray-tracing instance gather. Engine source lines captured from the log callstack;
  not traced into PinWright source beyond identifying `model.compile` as the trigger.
