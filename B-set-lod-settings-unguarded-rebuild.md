---
id: B-set-lod-settings-unguarded-rebuild
title: "geometry.set_lod_settings rebuilds a live UStaticMesh without the safe-point or render-consumer guards model.compile uses for the same fatal render-state hazard"
status: OPEN
severity: High
category: bug
tags: [geometry, set-lod-settings, static-mesh, rebuild, render-thread, niagara, safe-point, crash]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# `geometry.set_lod_settings` rebuilds render data on an ordinary handler stack

> **SOURCE-ONLY.** The crash path is established by the in-tree guard and engine call sequence; it
> was not executed during this scan.

`LODCollisionHandler.cpp:188-213` loads an existing `UStaticMesh`, mutates one source model, then
calls `StaticMesh->Build()` and `PostEditChange()` in place. The handler does not scan for live
components that cache the mesh's render data and is not listed in
`Source/PinWright/Private/Dispatch/SafePoint.cpp` (that table contains `model.compile`, but no
`geometry.*` entry).

The tree already documents why both gates are required. `MeshRebuildRenderGuard.h` says an in-place
`UStaticMesh` rebuild can leave non-standard scene proxies, including Niagara mesh renderers,
holding freed render data; the next gather can hit a render-thread `check()`. It also flushes render
commands, which is unsafe when the RPC was reached from a named-thread pump. `ModelCompileHandler.cpp:
1252-1284` scans live consumers, wraps the entire compile/save in `FQuiesceScope`, and refuses if a
consumer cannot be quiesced. `set_lod_settings` performs the same class of rebuild without either
protection.

## What should happen

Add `geometry.set_lod_settings` to the shared safe-point table and reuse
`PinWrightMeshRebuild::ScanForStaleRenderStateConsumers` plus `FQuiesceScope` around the mutation,
build, post-edit notification, and save. Refuse before mutation when any consumer remains live, and
name those consumers in the error. A test should prove the method is in the safe-point registry and
that an unquiescable live consumer prevents the build.

**Workaround:** close or deactivate every component using the target mesh and issue the call only
while the editor is idle. This reduces the risk but does not replace the missing safe-point gate.

## Related

`B-geometry-generate-lods-no-disk-write` owns persistence for this verb, not rebuild safety.
`B-model-compile-live-niagara-mesh-renderer-raytracing-assert` supplies the already-landed guard.

## History
- `#1-source-pattern-scan` `OPEN` reporter — `geometry.set_lod_settings` calls `UStaticMesh::Build`/`PostEditChange` directly, is absent from the shared safe-point registry, and bypasses the live render-consumer quiesce used by `model.compile`. Source-only; no editor or build was run.
