---
id: B-asset-generate-lods-no-disk-write
title: "asset.generate_lods marks meshes dirty through McpSafeAssetSave but reports completion with no disk-persistence result"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, generate-lods, static-mesh, persistence, no-disk-write, false-success]
---

# `asset.generate_lods` loses its LOD chain on cold load unless another save follows

## What's wrong

After changing source models, `asset.generate_lods` calls `McpSafeAssetSave(Mesh)`
(`AssetWorkflowHandler.cpp:1201-1203`). That helper intentionally only calls
`MarkPackageDirty`/`AssetCreated` and writes no package (`Utils/AssetUtils.cpp:503-518`). The
response then says `success:true` and “LOD generation completed” with only `processed` and
`lodCount` (`AssetWorkflowHandler.cpp:1209-1213`). It has no `saveRequested`, `saved`,
`pendingFlush`, or `saveState` field.

Concrete failure: generate LODs, trust the success response, close without a separate save, and
the on-disk mesh reloads with its old LOD chain. `B-geometry-generate-lods-no-disk-write` covers a
different namespace and explicitly does not fix this legacy Asset verb.

## What it should do

Add `save` (default true), drain compilation, persist with
`SaveAssetToDiskReportingPresence`, and attach the standard save report per mesh. A dirty-only
choice must be explicit and reported as pending rather than completed.

## Workaround

Call `asset.save` for every processed mesh after compilation has finished.

## Fix

Root cause: `asset.generate_lods` used the mark-dirty-only `McpSafeAssetSave`, so a successful response did not prove that the generated LOD chain had reached disk. The handler now starts one request-wide bounded deadline before the guarded callback, triggers one silent StaticMesh build, waits through the shared compile pump, uses the shared durable save helper when requested, and gates each row's success on the measured save outcome. A timeout reports the explicit non-attempted save state and retains the affected render-guard components until any in-flight build reaches terminal state.

Files changed:

- `Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/AssetWorkflowHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Utils/AssetCompilePump.h`
- `Plugins/PinWright/Source/PinWright/Private/Utils/MeshRebuildRenderGuard.h`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Assets/TestGenerateLodsPersistenceAndCompile.cpp`
- `Plugins/PinWright/Docs/wiki-src/asset.md`

Test IDs:

- `PinWright.asset.generate_lods.SavesToDisk`
- `PinWright.asset.generate_lods.NotReportedCompleteWhileCompiling`

Deliberate non-changes: no changes to `McpSafeAssetSave` or shared save helpers; the compile-pump override is confined to `WITH_DEV_AUTOMATION_TESTS`. The existing `FQuiesceScope` now minimally retains only the affected components needed by its raw render-state contexts. Batch semantics, unrelated handlers, and the comparator's explicit `save:false` `notRequested` shape remain unchanged. No build, test, editor, MCP, or runtime verification was performed in this source-only pass.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the persistence catalog; no editor, build, test, or RPC run was performed.
- `#2-generate-lods-persistence-compile` `IN-REVIEW` developer — Changed `AssetWorkflowHandler.cpp` to wait for StaticMesh compilation, persist generated LODs, and report the shared save state; added the two production-handler regression tests.
