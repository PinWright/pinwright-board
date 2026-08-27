---
id: B-model-compile-live-niagara-mesh-renderer-raytracing-assert
title: "model.compile on a static mesh a live Niagara mesh renderer is drawing kills the editor in the ray-tracing gather"
status: IN-REVIEW
severity: Critical
category: bug
tags: [model, static-mesh, niagara, mesh-renderer, ray-tracing, editor-crash, render-thread, stale-reference, missing-guard, shared-editor]
encounters: 2
lastSeen: 2026-08-27T19:29:41+05:00
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

## Merged from `B-static-mesh-rebuild-crashes-live-niagara-mesh-renderer`

Everything in this section came from `B-static-mesh-rebuild-crashes-live-niagara-mesh-renderer`, an
independent report of this **same crash** filed 19 s later by a different agent in the same editor —
same log, same assert, same innermost frame, same `SM_Bubble` trigger, same frame 505 -> 506
timeline. That ticket was **merged into this one and deleted**; nothing below was discarded.

### A second reading of the mechanism: a stale cached LOD index

The rebuild reallocates the mesh's `RenderData->LODResources`; the renderer's cached LOD index is
now past the end of the array, and the first ray-tracing instance gather after the rebuild indexes
out of bounds. `check()` on the render thread is an unrecoverable `appError` — no error response, no
chance to recover, the process is gone for every agent in the editor.

This is a **different reading of the same `-1 into an array of size 0`** than `## Symptom` above,
which reads it as "the mesh has no render data at the instant the gather runs, so the
highest-available-LOD computation returns -1". Both agree the gather runs against an array the
rebuild invalidated. Which of the two it actually is decides the guard: a stale *index* is fixed by
a reregister, an *empty* LOD array needs the component fully deactivated across the rebuild. A fixer
should settle this before choosing, rather than assuming either reading.

### Why this is ours and not just an engine assert

The engine assert is the symptom; the missing guard is the defect, and it is the **same shape** as
`B-niagara-edit-with-open-asset-editor-slate-crash`: a mutating verb returns success against an
object that something else in the editor still holds a live, now-stale reference to, and the process
dies on the next redraw rather than at the call. There the stale holder was an open Slate title bar;
here it is a render-thread proxy. Neither verb checks.

Nothing in the mesh-build path looks for dependent Niagara renderers. The rebuild is exactly what
`model.compile` does on every iteration of an authoring loop, so on a level where any Niagara system
uses a mesh renderer **the hazard is armed permanently**, and it fires on *whatever redraws next* —
in this instance a `render.capture_open_level` issued by an unrelated agent, which is how it was
observed. That capture request got a dropped stream and no diagnosis; the log was the only way to
learn the capture was a bystander rather than the cause. Any diagnosis that starts from "which call
returned the error" will therefore blame the wrong verb.

### Additional fix directions

These extend `## Suggested fix` below rather than replacing it:

1. **`FlushRenderingCommands()` alone is not enough.** The cached LOD index is *state*, not a queued
   command. The guard has to be `FComponentReregisterContext` (or `DeactivateImmediate()` +
   reactivate) per affected `UNiagaraComponent`, around the rebuild.
2. **Name the typed refusal.** `NIAGARA_RENDERER_DEPENDS_ON_MESH`, listing the blocking components,
   so the caller gets the deactivate-first workflow explicitly instead of a dead editor.
3. **Report the dependency in `static_mesh.describe`.** "This mesh is referenced by N live Niagara
   mesh renderers" is cheap, read-only, and would let a careful caller avoid the whole class before
   touching anything.
4. **Put the guard at the shared build call, not in `model.compile`.** Check whether
   `model.compile`, `geometry.convert_to_static_mesh` and the `asset.save` path all reach the same
   `UStaticMesh` build; if they do, a guard in `model.compile` alone leaves the other two armed.

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

**The deactivate variant is now proven end-to-end on SHIPPED actors, 2026-08-27 (see `#5`).** The
preview-actor advice above does not cover the case that matters most: a `VFX_*` actor that ships in
the level is a live consumer too, and it cannot be deleted. Sequence that worked, on eleven live
consumers of `SM_Bubble`:

1. Enumerate every consumer first. `grep -al "SM_Bubble"` over `Content/` found exactly two Niagara
   systems, whose eleven placed actors `actor.find_by_class {className:"NiagaraActor"}` then named.
   Do this before touching anything — deleting your own preview actor is not enough.
2. `effect.deactivate_niagara` on all eleven. It stops spawning but does NOT kill live particles.
3. **Wait out the longest particle lifetime** — here `Lifetime Max` 34 s from the ambient system's
   `InitializeParticle`, so 50 s. This is the step that empties the renderer.
4. **Gate on a capture, not on the clock.** `render.capture_open_level` at a pose that normally
   shows a bubble column, read the PNG, and confirm zero particles draw. A frame with no particles
   is the only observable proof the renderers have nothing left to gather; the wait alone is an
   assumption. The capture is safe here precisely because the rebuild has not happened yet.
5. `model.compile`, then `effect.activate_niagara` on all eleven.

Editor survived (`Building static mesh SM_Bubble` -> `Built static mesh [0.01s]`, 108 -> 432
triangles, no assert). Cost is a ~1 min quiesce window in which the level renders without the
effect, which is cheap against an editor kill.

## Suggested fix

The assert is in engine code, so PinWright cannot fix it directly, but it can stop reaching it:

- In `model.compile`, before swapping render data, find `UNiagaraComponent`s in the open world whose
  renderers reference the target `UStaticMesh` and mark them inactive / re-register them around the
  rebuild (`FComponentReregisterContext` equivalent), the way the static-mesh editor's own rebuild
  path does.
- Failing that, refuse with a typed error naming the live Niagara actors, so the caller deletes them
  first instead of losing the editor. A refusal that names the blocking actors is strictly better
  than a crash, and matches the house rule that an error must say what was checked and what to do.

## Related

- `B-static-mesh-rebuild-crashes-live-niagara-mesh-renderer` (**merged into this ticket and
  deleted**) was the SAME DEFECT, filed independently by another agent in the same editor from the
  same crash: same log (`Saved/Logs/EAContentExamples58-backup-2026.08.27-14.29.41.log`), same
  assert (`Array.h:1339`, `-1 into an array of size 0`), same innermost frame
  (`FNiagaraRenderableStaticMesh::GetRayTraceLODModelData`), same trigger (`SM_Bubble` rebuilt on
  frame 505, render thread aborts on frame 506). The two were complementary, not redundant, so
  everything that ticket carried and this one did not — the stale-LOD-index reading of the assert,
  the "why this is ours" argument, and the three extra fix directions including that the guard
  belongs wherever the shared `UStaticMesh` build call lives rather than in `model.compile` alone —
  is preserved above under `## Merged from ...`. **There is now one ticket for this defect; fix it
  once, here.**
- `B-capture-asset-preview-no-safe-close-mode` (OPEN, Critical) — **this ticket is the cost of
  that ticket's recommended workaround.** Its fix #3 tells callers to stop using
  `render.capture_asset_preview` and prove assets from the level instead (`niagara.spawn_actor` /
  `actor.spawn` + `render.capture_open_level` + delete). Following that advice is exactly what
  leaves a live Niagara mesh renderer in the level, which is the precondition here. The guidance
  is still right; it is incomplete. It must also say *delete the preview actor before recompiling
  the mesh it draws* — otherwise it trades a capture-path crash for this one.

## History
- `#1-initial-repro` `OPEN` reporter — Hit while authoring `/Game/Atlantis/VFX/NS_Bubbles_Ambient`
  on the Atlantis map build. `SM_Bubble` was recompiled from `subdivisions=3` to `subdivisions=4` to
  fix a visibly hexagonal silhouette, while `PWTEST_Bub2` was live in the viewport rendering it
  through a Niagara mesh renderer. Compile reported success; editor died on the next render-thread
  frame in the ray-tracing instance gather. Engine source lines captured from the log callstack;
  not traced into PinWright source beyond identifying `model.compile` as the trigger.

- `#2-independent-log-confirmation` `OPEN` reporter — 2026-08-27, crash-forensics pass over all
  five of the day's editor kills (no repro run; reproducing this one costs everyone in the editor
  another crash). **Confirms #1 from the log alone, without using the reporter's knowledge of what
  they had called** — worth recording because one of this session's earlier crash attributions
  ("Slate stack exhaustion") was wrong and was only caught when someone re-checked it.

  Independently verified in `Saved/Logs/EAContentExamples58-backup-2026.08.27-14.29.41.log`:
  `LogStaticMesh: Building static mesh SM_Bubble` at `14.29.37:520` and `Built static mesh [0.00s]`
  at `:521`, package saved `:574`-`:590`, and `appError` at `:605` — **85 ms from rebuild to
  assert**, frame 505 to frame 506. Full 22-frame callstack matches the one in `## Symptom`
  exactly. Ray tracing confirmed live in this session rather than assumed: `LogConfig: Set CVar
  [[r.RayTracing:1]]` and `[[r.Lumen.HardwareRayTracing:1]]` at `14.20.43:081`, which is what puts
  `GetDynamicRayTracingInstances` on the render path at all.

  Two corrections/limits on the evidence, stated because they bound what the log can prove:

  - **`## Repro`'s "Evidence" line points at the wrong file.** It cites
    `Saved/Logs/EAContentExamples58.log`; that log was rotated by this crash and now begins at
    19:30:43 local, after the fact. The crash lives in the retained backup named above (which
    `## Environment` in the sibling ticket cites correctly).
  - **The log does not name the triggering RPC.** `model.compile` emits no
    `LogPinWrightSubsystem` line, and the last such line in the session is at `14.28.08:693`
    (`actor.delete: PWTEST_BubCount`), 89 s earlier. The attribution to `model.compile` rests on
    the `LogStaticMesh` build+save pair, which `model.compile` is the verb that produces — sound,
    but inferred, not read. Likewise the log proves *a* live `FNiagaraRendererMeshes` from the
    callstack, but does not name the actor; `PWTEST_Bub2` in #1 is the reporter's own knowledge.

  Also note `LogRendererCore: Warning: FlushRenderingCommands called recursively! 2 calls on the
  stack.` at `14.29.41:686`, immediately after the assert. That is the exact tell already
  documented for this verb in `Plugins/PinWright/Source/PinWright/Private/Dispatch/SafePoint.cpp`
  § H (lines 289-293) — so `model.compile` is *already* on the tick-unsafe list and the hazard
  fired anyway; see `B-safepoint-tick-gate-inert-on-simpletickobjects-path` for why that list did
  not protect it.

- `#3-merged-independent-bystander-report` `OPEN` reporter — 2026-08-27T19:29:41+05:00 (log UTC
  14:29:41), same editor, same crash. Originally filed as its own ticket
  `B-static-mesh-rebuild-crashes-live-niagara-mesh-renderer`; **merged into this ticket and that file
  deleted** during a duplicate-consolidation pass — its content is preserved in
  `## Merged from ...` above, not discarded. Observed as a bystander while measuring rock/rubble
  grounding on the same level: another agent's `SM_Bubble` rebuild was the trigger, and this
  reporter's own `render.capture_open_level` merely supplied the redraw that fired it — the capture
  returned a dropped stream with no diagnosis. Not reproduced deliberately. Timeline and callstack
  read from `Saved/Logs/EAContentExamples58-backup-2026.08.27-14.29.41.log`; ray tracing confirmed
  active in the editor viewport.

- `#4-quiesce-render-consumers-around-in-place-rebuild` `IN-REVIEW` developer — `model.compile` no
  longer rebuilds a loaded mesh underneath a scene proxy that cached its render data. New
  header `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Model/MeshRebuildRenderGuard.h`
  carries the whole guard: a table of the component classes the engine does NOT reregister around a
  `UStaticMesh` build (`/Script/Niagara.NiagaraComponent`, resolved by CLASS PATH so
  `PinWrightGeometry` gains no link dependency on Niagara and a Niagara-less host simply finds no
  candidates), a `TObjectIterator` scan over every world — not just the editor world, since an
  asset-editor preview scene gathers ray-tracing instances too — for registered instances that hold
  render state, and an RAII `FQuiesceScope` that flushes, destroys their render state, flushes again
  (`DestroyRenderState_Concurrent` only ENQUEUES the teardown, so the second flush is what stops the
  rebuild landing while the render thread is still gathering), and recreates it on scope exit.
  `Source/PinWrightGeometry/Private/Handlers/Model/ModelCompileHandler.cpp` wraps the
  `FPwModelCompiler::Compile` call in that scope so the render state stays down across BOTH the build
  and the save, and arms it only when `FindObject` shows the output path already loaded — only a
  rebuild IN PLACE can strand a proxy, so a first-time compile scans nothing and pays nothing.
  The scope MEASURES its own result rather than assuming it: every component is read back after the
  attempt, and if any still holds render state (or was unreachable and could not be touched at all)
  the handler refuses with the new `ERR_MESH_REBUILD_CONSUMER_NOT_QUIESCABLE`, naming the components
  and building nothing — per `## Suggested fix`, a refusal that names the blockers beats an editor
  kill. The `overwrite` doc string's "being referenced is never a reason a compile is refused" now
  states that one exception instead of being quietly false. Regression tests
  `PinWright.Model.RebuildRenderGuard.QuiescedConsumersLoseAndRegainRenderState` and
  `PinWright.Model.RebuildRenderGuard.NiagaraMeshRendererIsAScannedCandidateClass` in
  `Source/PinWrightGeometry/Private/Tests/Model/TestModelRebuildRenderGuard.cpp` — the crash itself
  cannot be reproduced from automation (an `appError` on the render thread kills the suite host), so
  they assert the guard: that the scan finds a live proxy-holding component of a candidate class,
  that the scope strips its render state for the duration and restores it on exit with nothing left
  unquiesced, and that the shipped class table names `UNiagaraComponent` while excluding
  `UStaticMeshComponent` (which `UStaticMesh::PostEditChange` already reregisters — the reason a
  placed static-mesh actor survived the rebuild and the Niagara renderer did not).
  **`B-static-mesh-rebuild-crashes-live-niagara-mesh-renderer` is the same defect and this single
  change fixes both.** That ticket was assigned to this fix as a second file to flip, but a
  concurrent duplicate-consolidation pass merged it into this one and deleted its file (see `#3` and
  `## Merged from ...`) while this fix was in flight; its file was NOT recreated, so this bullet is
  the IN-REVIEW record for both. Not compiled or run here — the orchestrator owns builds — and
  `ERR_MESH_REBUILD_CONSUMER_NOT_QUIESCABLE` must be appended to `ErrorCodes.h` before this compiles.

- `#5-guard-absent-from-a-second-checkout-manual-quiesce-worked` `IN-REVIEW` reporter —
  2026-08-27, `EAContentExamples58`, raising `SM_Bubble` from `subdivisions=4` to `7` to kill a
  faceted bubble silhouette. **No crash: the hazard was avoided, not hit.** Two things worth
  recording.

  **`#4`'s guard is NOT in this checkout, so this tree is still armed.** Verified before relying on
  it, per the house rule that a board claim must be confirmed against the tree in hand:
  `Source/PinWrightGeometry/Private/Handlers/Model/MeshRebuildRenderGuard.h` does not exist here,
  `grep -rn ERR_MESH_REBUILD_CONSUMER_NOT_QUIESCABLE Source/` returns nothing, and
  `Binaries/Win64/UnrealEditor-PinWrightGeometry.dll` is dated 2026-08-26 19:13 — older than the
  fix. `#4` says "not compiled or run here"; it was written against a different one of the three
  hosts that share this board. **A reader of this ticket must not assume `model.compile` is
  guarded because `#4` is IN-REVIEW.** Until the fix is built and shipped to a given host, the
  manual sequence in `## Workaround` is the only protection there.

  **The workaround needed extending before it was usable.** As written it addressed *preview*
  actors and told the caller to delete them. The live consumers here were eleven shipped `VFX_*`
  actors — placed, density-tuned content that must not be deleted or re-seated — so the delete
  path did not apply and the deactivate path had to carry the whole job. What made it safe was
  gating on a capture rather than on the elapsed wait: `effect.deactivate_niagara` leaves particles
  alive for their full lifetime, so "deactivated" and "not drawing" are up to 34 s apart on this
  content, and only the frame proves which one you have. That sequence is now written into
  `## Workaround` above.

  This also sharpens `## Suggested fix` #3 (report the dependency in `static_mesh.describe`): the
  enumeration step had to be done with `grep` over `.uasset` bytes plus `actor.find_by_class`,
  because no read verb answers "who is drawing this mesh right now". `static_mesh.describe` on
  `/Game/Atlantis/Meshes/SM_Bubble` returns bounds, LODs and materials and says nothing about the
  eleven live Niagara components that make recompiling it fatal. That read-only field would have
  replaced the whole manual enumeration.
