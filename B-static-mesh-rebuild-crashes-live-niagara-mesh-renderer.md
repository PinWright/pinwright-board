---
id: B-static-mesh-rebuild-crashes-live-niagara-mesh-renderer
title: "Rebuilding a static mesh that a live Niagara mesh renderer references kills the render thread one frame later — `FNiagaraRenderableStaticMesh::GetRayTraceLODModelData` indexes the LOD array that the rebuild just reallocated"
status: OPEN
severity: Critical
category: bug
tags: [niagara, static-mesh, model-compile, crash, editor-kill, render-thread, ray-tracing, mesh-renderer, stale-reference, missing-guard, multi-agent, shared-editor]
encounters: 1
lastSeen: 2026-08-27T19:29:41+05:00
---

# A mesh rebuild and a live Niagara mesh renderer are 15 ms apart, and the second one dies

```
[14.29.37:520][505] LogStaticMesh: Display: Building static mesh SM_Bubble (Required Memory Estimate: 0.178148 MB)...
[14.29.37:521][505] LogStaticMesh: Built static mesh [0.00s] /Game/Atlantis/Meshes/SM_Bubble.SM_Bubble
[14.29.37:574][505] LogFileHelpers: Saving Package: /Game/Atlantis/Meshes/SM_Bubble
[14.29.37:590][505] LogFileHelpers: InternalPromptForCheckoutAndSave took 67.834 ms
[14.29.37:605][506] LogWindows: Error: appError called: Assertion failed: (Index >= 0) & (Index < ArrayNum)
                                 [File:Runtime/Core/Public/Containers/Array.h] [Line: 1339]
```

Frame 505 rebuilds and saves the mesh. Frame **506** — the very next one — aborts on the render
thread:

```
FNiagaraRenderableStaticMesh::GetRayTraceLODModelData()   NiagaraMeshRendererProperties.cpp:234
FNiagaraRendererMeshes::GetDynamicRayTracingInstances()   NiagaraRendererMeshes.cpp:1676
FNiagaraSystemRenderData::GetDynamicRayTracingInstances() NiagaraSystemRenderData.cpp:253
RayTracing::FDynamicRayTracingInstancesContext::GatherDynamicRayTracingInstances_Internal()
RayTracing::FinishGatherInstances()
FDeferredShadingSceneRenderer::Render()
RenderViewFamily_RenderThread()
```

A `UNiagaraComponent` in the level has a mesh renderer pointing at `SM_Bubble`. The rebuild
reallocates that mesh's `RenderData->LODResources`; the renderer's cached LOD index is now past the
end of the array, and the first ray-tracing instance gather after the rebuild indexes out of bounds.
`check()` on the render thread is an unrecoverable `appError` — no error response, no chance to
recover, the process is gone for every agent in the editor.

## Why this is ours and not just an engine assert

The engine assert is the symptom; the missing guard is the defect, and it is the **same shape** as
`B-niagara-edit-with-open-asset-editor-slate-crash`: a mutating verb returns success against an
object that something else in the editor still holds a live, now-stale reference to, and the process
dies on the next redraw rather than at the call. There the stale holder was an open Slate title bar;
here it is a render-thread proxy. Neither verb checks.

Nothing in the mesh-build path looks for dependent Niagara renderers. The rebuild is exactly what
`model.compile` does on every iteration of an authoring loop, so on a level where any Niagara system
uses a mesh renderer the hazard is armed permanently, and it fires on **whatever redraws next** — in
this instance a `render.capture_open_level` from an unrelated agent, which is how it was observed.
The capture request got a dropped stream and no diagnosis; the log was the only way to learn that
the capture was a bystander rather than the cause.

## Suggested fix

1. **Guard the rebuild.** Before building/saving a `UStaticMesh`, enumerate `UNiagaraComponent`s in
   the editor world whose system has a `UNiagaraMeshRendererProperties` referencing that mesh, and
   wrap the rebuild in `FComponentReregisterContext` (or `DeactivateImmediate()` + reactivate) for
   each. `FlushRenderingCommands()` alone is not enough — the cached LOD index is state, not a queued
   command.
2. **Or refuse and say so**, the way the conventions ask (`Never fake-success`): a typed
   `NIAGARA_RENDERER_DEPENDS_ON_MESH` naming the components is far better than a dead editor, and it
   gives the caller the deactivate-first workflow explicitly.
3. **Report the dependency in `static_mesh.describe`.** "This mesh is referenced by N live Niagara
   mesh renderers" is cheap and would let a careful caller avoid the whole class.

Worth checking whether `model.compile`, `geometry.convert_to_static_mesh` and the `asset.save` path
all reach the same build call, since the guard belongs wherever that is rather than in one handler.

## Impact

Critical, and the second editor kill from this session's Niagara surface (see
`B-niagara-create-node-early-return-before-finalize-crash`, filed the same hour). Five crashes today
in one shared editor with four agents in it; each one loses every unsaved level and asset edit in
flight, for everybody. The blast radius is wider than the caller: the agent that ran the rebuild got
its save committed, and the agent that merely asked for a screenshot got the crash.

## Environment

UE 5.8, `EAContentExamples58`, `/Game/Maps/Atlantis`, 2026-08-27T19:29:41+05:00 (log UTC 14:29:41).
Ray tracing active in the editor viewport. Full log retained at
`Saved/Logs/EAContentExamples58-backup-2026.08.27-14.29.41.log`.

## History

- `#1-reported` `OPEN` reporter — Observed as a bystander while measuring rock/rubble grounding on
  the same level; another agent's `SM_Bubble` rebuild is the trigger, my `render.capture_open_level`
  merely supplied the redraw. Not reproduced deliberately — reproducing it costs everyone in the
  editor another crash. Timeline and callstack read from the retained backup log.
