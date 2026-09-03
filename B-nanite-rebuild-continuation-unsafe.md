---
id: B-nanite-rebuild-continuation-unsafe
title: "render.nanite_rebuild_mesh runs its deferred Build and compilation drain outside the safe-point gate"
status: OPEN
severity: High
category: bug
tags: [render, nanite, static-mesh, safepoint, jobs, reentrancy]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# The Nanite rebuild continuation bypasses the safe-point gate

## What happens

`render.nanite_rebuild_mesh` binds its job by posting another
`AsyncTask(ENamedThreads::GameThread)` and starts the retained state from that later
continuation (`Source/PinWright/Private/Handlers/Render/RenderHandler.cpp:2463-2473`).
`FState::Start()` calls `UStaticMesh::Build()`
(`RenderHandler.cpp:2269-2294`, build at `:2277`), while the ticker completion path
calls `FStaticMeshCompilingManager::FinishCompilation()`
(`RenderHandler.cpp:2347-2357`). Neither continuation is routed through
`PinWrightSafePoint::RunAtSafePoint`.

## Why it matters

Static-mesh build and compilation drain can pump render/editor work after the
original dispatch stack and its request guard have returned. That reopens the
reentrancy class responsible for editor crashes on sibling engine-pumping routes.
Severity is High: crash-class impact is discounted one level because this Nanite
route is narrow and no direct crash through it is recorded.

## What should happen

Route the job's engine-pumping start and completion work through a safe-point-aware
continuation while preserving asynchronous job semantics, cancellation, and request
lifetime. Add structural coverage that the Nanite continuation cannot bypass the
gate and an ordering test for queued RPCs around it.

## Workaround

Run the rebuild only in an otherwise idle editor with no concurrent RPC traffic.
This reduces exposure but does not create a safe-point guarantee.

## Related

- `B-blueprint-mutators-compile-ungated-tick-unsafe` — wave-6 safe-point ticket
  whose grouped review exposed this continuation.
- `B-nested-gamethread-marshal-defeats-tick-gate` — explicitly retained this job
  marshal and recorded the Nanite safe-point need as separate work.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed the deferred GameThread job start at `RenderHandler.cpp:2463-2473`, `UStaticMesh::Build` at `:2277`, and compilation drain at `:2347-2357`, with no `RunAtSafePoint` in this route. No rebuild, build, test, editor, or MCP call was run. Severity High because the work is crash-class engine pumping on a narrow route with no direct Nanite crash observation.
