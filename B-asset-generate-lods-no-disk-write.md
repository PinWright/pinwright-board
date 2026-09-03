---
id: B-asset-generate-lods-no-disk-write
title: "asset.generate_lods marks meshes dirty through McpSafeAssetSave but reports completion with no disk-persistence result"
status: OPEN
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

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the persistence catalog; no editor, build, test, or RPC run was performed.
