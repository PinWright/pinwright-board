---
id: B-nanite-rebuild-continuation-unsafe
title: "render.nanite_rebuild_mesh runs its deferred Build and compilation drain outside the safe-point gate"
status: IN-REVIEW
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
continuation (`Source/PinWright/Private/Handlers/Render/RenderHandler.cpp:2522`).
`FState::Start()` calls `UStaticMesh::Build()`
(`RenderHandler.cpp:2333`), while the ticker completion and cancellation paths
calls `FStaticMeshCompilingManager::FinishCompilation()`
(`RenderHandler.cpp:2413`) and `FinishCompilationForObjects()` (`:2474`). Neither continuation is routed through
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

## Fix

Wave 8 guarded `asset.nanite_rebuild_mesh`, but it still used an unbounded compilation drain, while the independent render job escaped through a raw GameThread task and later used the same unbounded drain shape. `RenderHandler.cpp` now reserves `DeferJobToSafePoint` synchronously from the `StartJob` bind, quiesces live StaticMesh/Niagara consumers for the whole rebuild, and pumps compilation only inside the retained request scope. Both routes now accept a bounded `timeoutSeconds` value (default and maximum 120), re-check the deadline after every pump, refuse to save on timeout, and retain the mesh/render guard through an observation-only ticker until compilation is terminal; `SafePoint.cpp` also lists the render verb, and the old nested-marshal allow-list entry was removed.

Files changed:
- `Source/PinWright/Private/Handlers/Render/RenderHandler.cpp`
- `Source/PinWright/Private/Handlers/Asset/AssetWorkflowHandler.cpp`
- `Source/PinWright/Private/Dispatch/SafePoint.cpp`
- `Source/PinWright/Private/Tests/Assets/TestNaniteRebuildMeshSavesToDisk.cpp`
- `Source/PinWright/Private/Tests/EditorOps/TestRenderNaniteRebuildMeshSavesToDisk.cpp`
- `Source/PinWright/Private/Tests/Infra/TestHandlerTickSafetyRatchet.cpp`
- `Docs/wiki-src/asset.md`
- `Docs/wiki-src/render.md`

Tests added/extended:
- `PinWright.render.nanite_rebuild_mesh.SafePointContinuation` calls the production handler through an `FRpcDispatcher` with a real in-memory mesh and registered render consumer; it forces a deterministic timeout and proves RPC ordering, no save, and guard retention/restoration.
- `PinWright.asset.nanite_rebuild_mesh.BoundedCompileWait` calls the production handler with the same real mesh/consumer shape and proves `timedOut:true`, no save, and guard retention/restoration.
- `PinWright.infra.tick_safety.HandlerHazardsStayGated` now requires the render job continuation, rejects reintroducing its raw GameThread marshal, and asserts the method-level table entry.
- Existing `PinWright.render.nanite_rebuild_mesh.SavesToDisk` remains the handler-level rebuild/persistence/cancellation coverage.

Successful persistence/readback payloads retain the existing shape; `timeoutSeconds` is the only new request parameter. No build, editor, MCP, or automation run was performed in this source-only worker pass.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed the deferred GameThread job start at `RenderHandler.cpp:2522`, `UStaticMesh::Build` at `:2333`, and compilation drains at `:2413` / `:2474`, with no `RunAtSafePoint` in this route. No rebuild, build, test, editor, or MCP call was run. Severity High because the work is crash-class engine pumping on a narrow route with no direct Nanite crash observation.
- `#2-retained-safe-pump` `IN-REVIEW` developer — Replaced the render job's raw GameThread continuation and unbounded drains with `DeferJobToSafePoint`, a retained render-consumer guard, and a bounded shared compile pump; added handler-ordering and structural ratchets under `PinWright.render.nanite_rebuild_mesh.SafePointContinuation` and `PinWright.infra.tick_safety.HandlerHazardsStayGated`. No build, editor, MCP, or automation run was performed.
- `#3-bounded-asset-nanite-wait` `IN-REVIEW` developer — Replaced `asset.nanite_rebuild_mesh`'s unbounded compiler drain with the same 120-second shared pump and retained-guard timeout cleanup, plus `PinWright.asset.nanite_rebuild_mesh.BoundedCompileWait`. Successful response fields and request parameters are unchanged; no build, editor, MCP, or automation run was performed.
- `#4-verifier-timeout-follow-up` `IN-REVIEW` developer — Re-checked the deadline after each pump, made invalid render meshes terminate with `INVALID_ASSET`, replaced post-timeout Nanite pumping with passive guard retention, exposed bounded `timeoutSeconds` on both routes, and made both Nanite regression tests exercise real meshes and render consumers. Corrected the `b4329838` source citations; no build, editor, MCP, or automation run was performed.
- `#5-suite-timeout-expectation` `IN-REVIEW` developer — The wave-9 suite proved the test-only pending override could outlive the tiny fixture's real StaticMesh compilation. Corrected `PinWright.asset.nanite_rebuild_mesh.BoundedCompileWait` to require the consumer to remain quiesced only while `UStaticMesh::IsCompiling()` is true, matching `asset.generate_lods` and the ticket's terminal-state contract; the handler was already correct and was not changed. No build, editor, MCP, or automation run was performed in this follow-up.
- `#6-scoped-render-timeout-expectation` `IN-REVIEW` developer — The scoped wave-9 log recorded the render fixture's real mesh build completing in 0.00 seconds before the test asserted, while its test-only pending override still forced the timeout branch. Corrected `PinWright.render.nanite_rebuild_mesh.SafePointContinuation` to enforce the same `real compilation active => consumer quiesced` predicate as the asset test; the render handler already moves the quiesce scope into a passive retained watcher and was not changed. No build, editor, MCP, or automation run was performed in this follow-up.
