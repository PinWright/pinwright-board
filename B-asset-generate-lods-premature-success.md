---
id: B-asset-generate-lods-premature-success
title: "asset.generate_lods says generation completed before asynchronous StaticMesh compilation finishes"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, generate-lods, static-mesh, async-compile, premature-success, false-success]
---

# `asset.generate_lods` returns before the LOD build reaches a terminal state

## What's wrong

The handler calls `Mesh->Build()` and `PostEditChange()` at
`AssetWorkflowHandler.cpp:1201-1202`, increments `SuccessCount`, and responds “LOD generation
completed” at `:1205-1213`. It never calls `FStaticMeshCompilingManager::FinishCompilation` and
does not return a job ticket.

On UE 5.8, `UStaticMesh::Build` routes through `BatchBuild`; when async compilation is allowed it
creates `FStaticMeshAsyncBuildTask`, registers the mesh with the compiling manager, and returns
immediately (`StaticMeshBuild.cpp:249-277`, `:354-372`). A subsequent read/save can therefore race
the build, and a background build failure cannot change the already-green response. The sibling
Nanite and bake-transform handlers explicitly drain compilation before readback/save.

## What it should do

Finish compilation for every processed mesh before measuring, saving, and responding, and fail the
request if a mesh cannot reach a valid completed state. Alternatively expose a tracked job whose
terminal result owns the completion verdict.

## Workaround

Poll StaticMesh compilation externally before reading or saving; PinWright exposes no direct
per-mesh completion result for this verb.

## Fix

Root cause: `asset.generate_lods` treated `Build()` and `PostEditChange()` as terminal even though UE can leave the StaticMesh registered for asynchronous compilation, so source-model readback, save, and success accounting raced the build. The handler now starts one request-wide deadline before the guarded callback, uses the single silent `Build()` trigger, pumps compilation on the game thread until completion or that bounded timeout, publishes `timedOut`, and only then reads, saves, and marks the row successful. When a timeout leaves compilation in flight, the existing render guard retains its affected components until the manager reports terminal state.

Files changed:

- `Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/AssetWorkflowHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Utils/AssetCompilePump.h`
- `Plugins/PinWright/Source/PinWright/Private/Utils/MeshRebuildRenderGuard.h`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Assets/TestGenerateLodsPersistenceAndCompile.cpp`
- `Plugins/PinWright/Docs/wiki-src/asset.md`

Test IDs:

- `PinWright.asset.generate_lods.SavesToDisk`
- `PinWright.asset.generate_lods.NotReportedCompleteWhileCompiling`

Deliberate non-changes: no unbounded `FinishCompilation` call, no new job system, and no changes to production compile/save behavior, `environment.build` forwarding, unrelated handlers, or live PIE behavior; the compile-pump override is confined to `WITH_DEV_AUTOMATION_TESTS`. The existing render guard was minimally extended to retain only the affected components needed by its raw contexts; no expiry or unsafe cancellation was added. Existing batch/preflight semantics remain intact. No build, test, editor, MCP, or runtime verification was performed in this source-only pass.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation against UE 5.8 engine code; no editor, build, test, or RPC run was performed.
- `#2-generate-lods-persistence-compile` `IN-REVIEW` developer — Changed `AssetWorkflowHandler.cpp` to wait for terminal StaticMesh compilation before readback/save/success and report timeouts; added the two production-handler regression tests.
