---
id: B-geometry-convert-static-mesh-no-disk-write
title: "geometry.convert_to_static_mesh reports created:true/exists:true but never writes the .uasset — the baked StaticMesh is dirty-in-memory only and vanishes on editor close / cold load"
status: IN-REVIEW
severity: Critical
category: bug
tags: [geometry, convert_to_static_mesh, static-mesh, save, no-disk-write, silent-failure, false-success, cold-load, asset-loss, create-new-static-mesh-asset]
encounters: 2
lastSeen: 2026-06-25T07:14:35Z
---

# geometry.convert_to_static_mesh's success payload (`created:true`, `exists:true`) is a lie — the baked StaticMesh is never flushed to disk and is gone on the next editor launch

`geometry.convert_to_static_mesh` (`MeshOpsHandler.cpp:366-446`) is the terminal
"bake this DynamicMesh into a real `/Game/...` StaticMesh asset" verb at the end
of every procedural-prop authoring chain. On success it returns:

```
{ actorName, assetPath, exists:true, created:true, class:"StaticMesh",
  triangleCount, vertexCount, nanite:false, recomputeNormals:true,
  recomputeTangents:true }
  message: "StaticMesh created from DynamicMesh"
```

**But no `.uasset` is ever written to disk.** The handler calls
`UGeometryScriptLibrary_CreateNewAssetFunctions::CreateNewStaticMeshAssetFromMesh(...)`
(`MeshOpsHandler.cpp:405-406`), which creates the package + `UStaticMesh` object
and registers it with the asset registry — so `asset.exists` and `asset.get`
both report it (registry hit, not disk), and the `exists:true`/`created:true`
echo (added by `E-geometry-convert-static-mesh-no-asset-echo`) reads `true`
straight off the in-memory `Target.Mesh`. **There is no `UPackage::Save` /
`SaveLoadedAssetThrottled` / `McpSafeAssetSave` / `editor.save_all` call anywhere
in the handler** (the success branch at `MeshOpsHandler.cpp:414-445` builds the
JSON and returns; lines 405-413 are the only asset-touching code). The package is
left dirty-in-memory only. After the editor closes (or a fuzz `git reset --hard`)
the asset is gone, with no error and no `pendingFlush`/disk-presence signal ever
surfaced.

This is the geometry-bake analog of `B-niagara-save-no-disk-write` (niagara
path), `B-metasound-create-save-no-disk-write` (MetaSound path), and
`B-create-level-saved-true-no-umap` (level path): a create verb that reports
persistence success backed only by an in-memory/registry create, masked by a
registry-based `exists:true`. It is **worse** than those: there is no `save`
param to toggle and no `existsAfter`/`pendingFlush` field at all — the response
asserts `created:true` unconditionally, so a caller has *no* signal that the
bake is memory-only.

## Why it matters (Critical — asset loss on a normal path)

"Model a prop with the geometry helpers, then bake it to a StaticMesh I can drop
into levels" is the canonical, explicit purpose of this verb — it is the final
step of every procedural-prop workflow. Because `convert_to_static_mesh` answers
`created:true`/`exists:true` (and `asset.exists`/`asset.get` corroborate it from
the registry), an agent reasonably believes the asset is persisted and ends the
workflow. The entire model — box plate → central bore → four bolt-hole
subtractions → bevel → convex-decomposition collision → bake — is then **lost**
on the next editor launch, with `created:true` falsely asserted. `editor.save_all`
is the only way to actually flush the dirty package, and nothing tells the agent
that is required. This is silent data loss on the verb's primary path.

## Cold-load confirmation (the editor restart IS the replay)

A real headless editor cold-restart reopened the asset this attempt baked and the
load FAILED — the package is absent from disk on a fresh editor:

```
editor.quit {discard:true} -> {requested:true}; process exited in ~3s;
relaunched headless; MCP namespace index responded after ~60s.
Cold open /Game/GeneratedMeshes/SM_PipeFlange -> [ASSET_NOT_FOUND] Asset not found.
(SM_PipeFlange is a StaticMesh, non-Blueprint, so no blueprint.compile step
applied. The editor stayed up and reachable after the failed open.)
Outcome: load_failed (1/1 asset failed to open; editor stayed alive).
```

Filesystem corroboration on the host: there is no `Content/GeneratedMeshes/`
directory at all and no `SM_PipeFlange*` anywhere under `Content/` — the `.uasset`
was never written. The convert call's `created:true` was the only persistence
signal and it was false.

## Repro (verbatim from the attempt; replayable)

1. `geometry.create_box {name:"FlangePlate", ~200×200×30, @origin}` → ok.
2. `geometry.create_cylinder {name:"PipeBore", r45 h60 @origin}` → ok;
   `geometry.boolean_subtract {tool:PipeBore, target:FlangePlate, keepTool:false}` → ok.
3. four `geometry.create_cylinder {r12 h60 @±70,±70}` + four
   `geometry.boolean_subtract {... keepTool:false}` → all ok.
4. `geometry.bevel {FlangePlate, distance:3}` → ok (final 2276 verts / 4568 tris).
5. `geometry.generate_collision {FlangePlate, convex_decomposition}` → ok.
6. `geometry.convert_to_static_mesh {actorName:"FlangePlate", assetPath:"/Game/GeneratedMeshes/SM_PipeFlange"}`
   → `ok {assetPath:"/Game/GeneratedMeshes/SM_PipeFlange", exists:true, created:true, class:"StaticMesh", triangleCount:4568, ...}`.
7. `asset.exists {/Game/GeneratedMeshes/SM_PipeFlange}` → true (registry).
8. `asset.get {/Game/GeneratedMeshes/SM_PipeFlange}` → class=StaticMesh (registry).
9. **Editor cold restart → open `/Game/GeneratedMeshes/SM_PipeFlange` → `[ASSET_NOT_FOUND]`.** The `.uasset` was never on disk.

## What it should do

Mirror the accepted sibling fixes (`B-niagara-save-no-disk-write` #2,
`B-create-level-saved-true-no-umap` #2): persist for real on the geometry bake
path and report disk presence honestly, leaving the shared corruption-driven
`McpSafeAssetSave` no-op untouched.

- After `CreateNewStaticMeshAssetFromMesh` returns `Success`, route the new
  package through the in-tree real-save helper `SaveLoadedAssetThrottled`
  (`Utils/AssetUtils.cpp`, `UEditorAssetLibrary::SaveLoadedAsset` behind the
  Blueprint integrity gate) so the `.uasset` actually lands on disk. (A
  StaticMesh is not a Blueprint/widget, so the corruption history that drove the
  deferred mark-dirty for BP/SCS/MetaSound does not apply here — a real save of a
  StaticMesh package is safe.)
- Probe on-disk presence (`IFileManager::FileSize(PackageFilename)`) and gate the
  reported persistence through the existing shared predicate
  `ShouldTreatAssetSaveAsSuccess(bSaveReportedSuccess, bFileExistsOnDisk)` (added
  for the niagara fix). Report `created`/`exists` (or a new `savedToDisk`) as
  `true` only when the `.uasset` is actually on disk; when it is dirty-only,
  report it `false` plus a `pendingFlush:true` signal so the caller knows an
  `editor.save_all` is required — never an unconditional `created:true` for a
  memory-only package.

A docs-only floor (note on `docs/wiki-src/geometry.md` that
`convert_to_static_mesh` leaves the baked package dirty-in-memory and that the
asset is lost on editor close unless `editor.save_all` is called) would at least
make the requirement discoverable, but the behavior fix is preferred — the bake
verb's *only* purpose is to produce a reusable on-disk asset.

**Workaround:** call `editor.save_all` immediately after every
`geometry.convert_to_static_mesh` to flush the dirty package to disk before the
editor closes; do not trust `created:true`/`exists:true`/`asset.exists` as proof
the `.uasset` is persisted.

## Dedup

Distinct from `E-geometry-convert-static-mesh-no-asset-echo` (IN-REVIEW,
ergonomic) — that ticket added the `exists:true`/`created:true`/`triangleCount`
confirmation **echo** to collapse the bake+confirm readback; it never claimed (or
fixed) that the asset is on disk. This ticket is that very echo being *false* on
cold load: the confirmation block reports `created:true` for a package the handler
never flushed. The sibling save-fidelity tickets (`B-niagara-save-no-disk-write`,
`B-metasound-create-save-no-disk-write`, `B-create-level-saved-true-no-umap`) each
explicitly scope their fix to their own handler path and do not touch the geometry
bake path — there is no static-mesh / geometry-convert disk-write ticket, so this
fills the gap. Not the boolean-offset bug (`B-boolean-subtract-ignores-tool-offset`)
nor the orientation-axes docs gap.

## History
- `#1-initial-repro` `OPEN` reporter — Cold-load corruption confirmed by a real headless editor restart: after a full procedural-prop build (box plate → r45 central bore → four r12 corner bolt-hole boolean-subtracts → bevel dist 3 → convex_decomposition collision), `geometry.convert_to_static_mesh {actorName:"FlangePlate", assetPath:"/Game/GeneratedMeshes/SM_PipeFlange"}` returned `{exists:true, created:true, class:"StaticMesh", triangleCount:4568}` and `asset.exists`/`asset.get` corroborated it from the registry — but on cold restart the open failed with `[ASSET_NOT_FOUND]` and the `.uasset` is absent on disk (no `Content/GeneratedMeshes/` directory exists). Root cause (verified in source): `MeshOpsHandler.cpp:366-446` calls `UGeometryScriptLibrary_CreateNewAssetFunctions::CreateNewStaticMeshAssetFromMesh` (`:405-406`) — which creates + registers the package but does not flush it — then builds the success JSON and returns (`:414-445`) with NO `UPackage::Save`/`SaveLoadedAssetThrottled`/`McpSafeAssetSave`/`editor.save_all` anywhere. The package is dirty-in-memory only; `exists`/`created` are read off the in-memory `Target.Mesh` and the registry, not disk. Same defect family as `B-niagara-save-no-disk-write` / `B-metasound-create-save-no-disk-write` / `B-create-level-saved-true-no-umap` but on the geometry bake path (no `save` param, no `pendingFlush` signal, `created:true` unconditional). Critical: the verb's entire purpose is producing a reusable on-disk asset, and on a normal completion the whole model is silently lost on the next editor launch with `created:true` falsely asserted. Fix: route the baked package through `SaveLoadedAssetThrottled`, probe `IFileManager::FileSize`, gate persistence via the existing `ShouldTreatAssetSaveAsSuccess` predicate, and report `pendingFlush:true` when dirty-only — mirroring the accepted niagara/level sibling fixes, leaving the shared `McpSafeAssetSave` no-op untouched. Dedup: distinct from `E-geometry-convert-static-mesh-no-asset-echo` (which ADDED the now-false `created:true` echo but never addressed disk persistence) and from the per-path niagara/metasound/level save tickets (none cover geometry). Workaround: `editor.save_all` immediately after every convert.
- `#2-additional-cold-load` `OPEN` reporter — Additional evidence (second distinct prop, same defect): a hollow-stone-planter build (`geometry.create_cylinder PlanterOuter r60 h80 seg24 @z0` + `create_cylinder PlanterInner r48 h80 seg24 @z15` → `boolean_subtract {target:PlanterOuter, tool:PlanterInner, keepTool:false}` 144→336 tris → `bevel distance:2` 672 tris → `auto_uv` → `generate_collision convex` → `convert_to_static_mesh {actorName:"PlanterOuter", assetPath:"/Game/GeneratedMeshes/StonePlanter"}`) returned `{created:true, ...672t}` and `asset.exists {/Game/GeneratedMeshes/StonePlanter}` reported `exists:true` from the registry. On a real headless editor cold-restart (`editor.quit{discard:true}` → `{requested:true, discarded:true, dirtyCount:2}`, process exited cleanly in ~3s, relaunched detached headless WITHOUT any git reset/clean/lfs-checkout so saved assets would persist, MCP came back up), `editor.open_asset {/Game/GeneratedMeshes/StonePlanter}` → `[ASSET_NOT_FOUND] Asset not found`; the editor stayed up and reachable (namespace index responded before and after), so load_failed not editor_crash. Filesystem corroboration: no `Content/GeneratedMeshes/` folder and no `StonePlanter*` package anywhere in the repo tree — the baked `.uasset` was never written to disk. Confirms #1's root cause holds for a different asset path/topology (StaticMesh, not BP/Widget, so compile N/A); `dirtyCount:2` at quit shows the package was still dirty-in-memory at editor close. Same fix.
- `#3-fix` `IN-REVIEW` developer — Fixed the no-disk-write defect on the geometry bake path. `geometry.convert_to_static_mesh` (`MeshOpsHandler.cpp` success branch) now captures the `UStaticMesh*` returned by `CreateNewStaticMeshAssetFromMesh` and routes it through the consolidated real-save wrapper `SaveAssetToDiskReportingPresence(BakedMesh, /*bForce=*/true, &package, &sizeBytes)` (forced `SaveLoadedAsset` + `IFileManager::FileSize` disk probe gated by `ShouldTreatAssetSaveAsSuccess`) instead of returning straight off the in-memory create. `exists`/`created` are now gated on the `.uasset` actually landing on disk (`bSavedToDisk`) rather than hardcoded `true`; the response also carries `package`/`sizeBytes` and the standard `AddAssetSaveReport` verdict (`saveRequested`/`saved`/`pendingFlush`) so a dirty-only outcome reports `created:false` + `pendingFlush:true` rather than a false-success `created:true`. Implemented via the established `SaveAssetToDiskReportingPresence`/`AddAssetSaveReport` wrapper the niagara/level/metasound/audio siblings adopted (the ticket's older primitive-level text — manual `SaveLoadedAssetThrottled`+`FileSize`+`ShouldTreatAssetSaveAsSuccess` — predates that consolidation; behaviour is identical). Left the shared corruption-driven `McpSafeAssetSave` no-op untouched (a StaticMesh is not a BP/widget). Files: `Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp` (added `#include "Engine/StaticMesh.h"` + `"Utils/AssetUtils.h"`). Regression test: `Source/PinWright/Private/Tests/Geometry/TestGeometryConvertStaticMeshSavesToDisk.cpp` (`PinWright.geometry.convert_to_static_mesh.SavesToDisk`) — bakes a real DynamicMeshActor through the live dispatcher and probes `IFileManager::FileSize` on the resolved package filename, asserting the `.uasset` is genuinely on disk plus `saved:true`/`sizeBytes>0`/no `pendingFlush`; reverting the save call leaves the package dirty-only so no file lands and the probe fails. Scope note (not addressed here, per per-path convention): `geometry.convert_to_nanite` (`MeshOpsHandler.cpp` convert-to-nanite handler) and `geometry.generate_lods` (`LODCollisionHandler.cpp`, which calls the mark-dirty no-op `McpSafeAssetSave`) share the identical no-disk-write gap and warrant a companion ticket.
