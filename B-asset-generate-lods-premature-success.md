---
id: B-asset-generate-lods-premature-success
title: "asset.generate_lods says generation completed before asynchronous StaticMesh compilation finishes"
status: OPEN
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

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation against UE 5.8 engine code; no editor, build, test, or RPC run was performed.
