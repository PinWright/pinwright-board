---
id: B-geometry-generate-lods-no-disk-write
title: "geometry.generate_lods (and siblings set_lod_settings / set_lod_screen_sizes) mutate a StaticMesh's LOD chain but only McpSafeAssetSave (mark-dirty no-op) it and report a hardcoded existsAfter:true — the LOD data is dirty-in-memory only and lost on cold load, with no saved/pendingFlush signal"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, generate_lods, set_lod_settings, set_lod_screen_sizes, static-mesh, lod, save, no-disk-write, mcp-safe-asset-save, silent-failure, false-success, cold-load, persistence]
encounters: 1
lastSeen: 2026-07-02T17:42:04.1496265+03:00
---

# `geometry.generate_lods` builds an N-LOD chain into a StaticMesh but never flushes it to disk — `existsAfter:true` is hardcoded and there is no `saved`/`pendingFlush` signal, so the LODs are lost on the next editor launch unless a separate `asset.save` is fired

`geometry.generate_lods` (`LODCollisionHandler.cpp:175-302`) is the "give this
baked StaticMesh a LOD ladder so it fades at distance" verb at the tail of every
procedural-prop-to-game-ready workflow. It really does the work: it
`SetNumSourceModels(LODCount)`, configures per-LOD reduction, calls
`StaticMesh->Build()` + `PostEditChange()` — a genuine mutation that adds LOD
bytes to the asset. **But the only persistence call in the handler is
`McpSafeAssetSave(StaticMesh)` (`LODCollisionHandler.cpp:291`), which does NOT
write the `.uasset`:**

```cpp
bool McpSafeAssetSave(UObject* Asset)   // AssetUtils.cpp:214-226
{
    // UE 5.7+ Fix: Do not immediately save newly created assets to disk.
    // ... Instead, mark the package dirty and notify the asset registry.
    Asset->MarkPackageDirty();
    FAssetRegistryModule::AssetCreated(Asset);
    return true;
}
```

So the LOD chain is left **dirty-in-memory only**. The success response is built
by `AddAssetVerification(Result, StaticMesh)` (`LODCollisionHandler.cpp:298`),
which sets `existsAfter` to a **hardcoded `true`** (`AssetUtils.cpp:1098` —
`Response->SetBoolField(TEXT("existsAfter"), true)`), a registry/memory check,
not a disk-presence probe. The response carries **no `saved`/`saveRequested`/
`pendingFlush`/`sizeBytes` field at all** — nothing tells the caller the LOD
data is unflushed. The in-tree save-fidelity comment says it plainly
(`AssetUtils.cpp:422-424`): *"helper McpSafeAssetSave returns true without
writing (deferred to dodge the bulkdata-corruption vector) … the on-disk file is
the only honest persistence signal."*

The two sibling LOD verbs in the same file share the identical no-op:
`geometry.set_lod_settings` (`LODCollisionHandler.cpp:360`) and
`geometry.set_lod_screen_sizes` (`:439`) both mutate the StaticMesh's LOD
settings and then only `McpSafeAssetSave(StaticMesh)` + `AddAssetVerification`.
So the whole LOD-authoring family in `LODCollisionHandler.cpp` leaves its work
dirty-in-memory with a hardcoded `existsAfter:true`.

## This is the companion ticket the convert-to-static-mesh fix predicted

`B-geometry-convert-static-mesh-no-disk-write` `#3-fix` (IN-REVIEW) fixed the
bake verb's no-disk-write by routing it through
`SaveAssetToDiskReportingPresence` and gating `created`/`exists` on real disk
presence — and its closing scope note reads verbatim: *"`geometry.generate_lods`
(`LODCollisionHandler.cpp`, which calls the mark-dirty no-op `McpSafeAssetSave`)
share[s] the identical no-disk-write gap and warrant[s] a companion ticket."*
This is that ticket, now with live replay evidence.

## Why it matters (High — silent LOD loss on a normal path)

"Bake a prop and give it a couple of LODs so it fades cleanly at distance" is the
canonical, explicit purpose of this verb — the last step of every
procedural-prop-to-game-ready chain. Because `generate_lods` answers
`existsAfter:true` with no persistence warning, a caller reasonably believes the
LOD'd asset is on disk and ends the workflow. On the next editor launch the LOD
chain is gone (the `Build()`-produced source models were never serialized), with
`existsAfter:true` having falsely implied persistence. It is the same
false-success save-fidelity class as `B-geometry-convert-static-mesh-no-disk-write`,
`B-niagara-save-no-disk-write`, `B-metasound-create-save-no-disk-write`,
`B-audio-create-save-no-disk-write`, `B-material-authoring-save-no-disk-write`
and `B-create-level-saved-true-no-umap`. Filed High rather than Critical only
because the LOD-authoring step almost always runs immediately after
`convert_to_static_mesh` (whose #3-fix now DOES save the base mesh to disk), so
the base asset survives and only the LOD ladder is lost — still silent wrong
persistence on the verb's primary path, and the LOD-add is the entire point of
the call.

## Live replay evidence (this task — geometry.simplify_mesh "SM_RockBoulder" boulder-optimization build, 22 calls, outcome clean, friction:"none")

Struggle audit of the boulder-prop optimization build (10 wiki-nav reads + 12
RPCs, all first-try clean, no retries/`python.execute`; the per-finding judge
filed nothing, `filed_id` empty). The bake+LOD tail, quoted verbatim from the
attempt transcript:

- `geometry.convert_to_static_mesh {actorName:RockBoulder → /Game/GeneratedMeshes/SM_RockBoulder}` → `{exists:true, created:true, saveRequested:true, saved:true, sizeBytes:764033, triangleCount:23064, ...}` — the base mesh IS on disk (`#3-fix` of the convert ticket working; `saved:true` + `sizeBytes:764033`).
- `geometry.generate_lods {assetPath:/Game/GeneratedMeshes/SM_RockBoulder, lodCount:4}` → `{"assetPath":"/Game/GeneratedMeshes/SM_RockBoulder","lodCount":4,"triangles":23064,"assetName":"SM_RockBoulder","existsAfter":true,"assetClass":"StaticMesh"}` — **no `saved`/`pendingFlush`/`sizeBytes`; `existsAfter:true` hardcoded.**
- The agent recognized the gap and, per the workflow's persistence rule, fired a separate `asset.save {assetPath:/Game/GeneratedMeshes/SM_RockBoulder, force:true}` → `{"saved":true,"sizeBytes":773193}`. Its own reasoning, verbatim: *"Let me save the asset with force to persist the LOD chain to disk, per the persistence rule."*
- `static_mesh.describe {SM_RockBoulder}` → `lods:4, trianglesByLod:[23064,11532,5766,2882]` — the LODs are present.

The **`sizeBytes` grew from 764033 (post-convert) to 773193 (post-explicit-save)
— a +9160-byte delta that is exactly the LOD chain data**. That delta is
direct, replayable proof that `generate_lods` had NOT flushed the LODs to disk
when it returned `existsAfter:true`; only the subsequent forced `asset.save`
wrote them. Had the agent trusted `existsAfter:true` and skipped the manual save
(as a normal user reasonably would, seeing an existence-confirming success), the
4-LOD chain would have been lost on the next cold load. This is pure silent
persistence overhead surfaced by the struggle audit, not any error — every call
succeeded.

## What it should do

Mirror the accepted sibling fixes (`B-geometry-convert-static-mesh-no-disk-write`
`#3-fix`, `B-niagara-save-no-disk-write` `#2`): after `Build()`/`PostEditChange()`,
route the StaticMesh through the consolidated real-save wrapper
`SaveAssetToDiskReportingPresence(StaticMesh, /*bForce=*/true, &package, &sizeBytes)`
(forced `SaveLoadedAsset` + `IFileManager::FileSize` disk probe gated by
`ShouldTreatAssetSaveAsSuccess`) instead of the mark-dirty-only `McpSafeAssetSave`,
and report persistence honestly via the standard `AddAssetSaveReport` verdict
(`saveRequested`/`saved`/`pendingFlush`/`sizeBytes`) so a dirty-only outcome
reports `saved:false` + `pendingFlush:true` rather than a bare hardcoded
`existsAfter:true`. A StaticMesh is not a Blueprint/widget, so the
corruption-driven deferral that `McpSafeAssetSave` exists to dodge does not apply
here — a real save of a StaticMesh package is safe (this is exactly the argument
`B-geometry-convert-static-mesh-no-disk-write` `#3-fix` used for the same asset
class). Extend the same treatment to `set_lod_settings` (`:360`) and
`set_lod_screen_sizes` (`:439`), which share the identical no-op.

A docs-only floor (note on `docs/wiki-src/geometry.md` that `generate_lods` /
`set_lod_settings` / `set_lod_screen_sizes` leave the StaticMesh dirty-in-memory
and that the LOD data is lost on editor close unless `asset.save {force:true}` is
called) would at least make the requirement discoverable, but the behavior fix
is preferred — the LOD verbs' whole purpose is to produce a persisted, reusable
LOD'd asset.

**Workaround:** call `asset.save {assetPath, force:true}` immediately after
`geometry.generate_lods` (and after any `set_lod_settings` /
`set_lod_screen_sizes`) to flush the LOD data to disk — exactly what this task
had to do; do not trust the response's `existsAfter:true` as proof the LODs are
on disk.

## Dedup

Distinct from `B-geometry-convert-static-mesh-no-disk-write` (IN-REVIEW) — that
ticket's `#3-fix` fixed the *bake* verb (`convert_to_static_mesh`) and its scope
note explicitly deferred the `generate_lods` no-disk-write to a companion ticket
(this one). Distinct from `E-generate-lods-landscapepath-misnomer` (IN-REVIEW),
which is about the `asset.generate_lods` param *name* (`landscapePath` vs
`assetPath`), a different method and a naming — not persistence — friction.
Distinct from `F-geometry-lod-settings-batch` (OPEN), which is the *call-count*
asymmetry of `set_lod_settings` having no batch form, not persistence. Distinct
from the per-path niagara/metasound/audio/material/level save-fidelity tickets,
none of which touch the geometry LOD path. No existing ticket covers
`geometry.generate_lods` / `set_lod_settings` / `set_lod_screen_sizes` disk
persistence.

severity rationale: impact=silent wrong persistence / asset-loss on a normal path (existsAfter:true implies persistence for a dirty-only LOD chain) × reach=every-session (LOD authoring is the tail of every game-ready-prop build) -> High (one level below the convert ticket's Critical because the base mesh is now saved by the convert #3-fix, so only the LOD ladder is lost, not the whole model)

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.simplify_mesh` "SM_RockBoulder" boulder-optimization build (22 calls: 10 wiki-nav + 12 RPCs, outcome clean, friction:"none", judge filed nothing, `filed_id` empty). PROCESS/persistence finding, and the companion ticket predicted by `B-geometry-convert-static-mesh-no-disk-write` `#3-fix`'s scope note: `geometry.generate_lods` (`LODCollisionHandler.cpp:175-302`) builds an N-LOD chain (`SetNumSourceModels`+`Build()`+`PostEditChange()`) but persists it only via the mark-dirty-only `McpSafeAssetSave` (`AssetUtils.cpp:214-226` — `MarkPackageDirty`+`AssetCreated`, NO disk write) and reports a hardcoded `existsAfter:true` (`AddAssetVerification`, `AssetUtils.cpp:1091-1100`) with no `saved`/`pendingFlush`/`sizeBytes`. Live replay proof: after `convert_to_static_mesh` saved the base mesh (`saved:true, sizeBytes:764033`), `generate_lods {lodCount:4}` returned `{lodCount:4, existsAfter:true}` with no persistence field; the agent then had to fire `asset.save {force:true}` (its reasoning: *"persist the LOD chain to disk, per the persistence rule"*) which grew `sizeBytes` 764033→773193 (+9160 bytes = the LOD data) — proving `generate_lods` had NOT flushed the LODs. `set_lod_settings` (`:360`) and `set_lod_screen_sizes` (`:439`) share the identical no-op. Fix: route through `SaveAssetToDiskReportingPresence` + `AddAssetSaveReport` (a StaticMesh is not a BP/widget, so a real save is safe — same argument as the convert `#3-fix`), report `saved`/`pendingFlush` honestly instead of a hardcoded `existsAfter:true`; extend to the two sibling LOD verbs. Docs floor: `docs/wiki-src/geometry.md`. Workaround: `asset.save {force:true}` after every LOD verb. Dedup: distinct from the convert-to-static-mesh no-disk-write ticket (this is its predicted companion), the `landscapePath` misnomer ticket (naming, `asset.generate_lods`), the `set_lod_settings` batch ticket (call-count), and the per-path niagara/metasound/audio/material/level save tickets (none touch geometry LODs).
- `#2-fix` `IN-REVIEW` developer — Verified every load-bearing claim against synced source: `LODCollisionHandler.cpp` `generate_lods` (:291), `set_lod_settings` (:360), `set_lod_screen_sizes` (:439) all persisted only via the mark-dirty no-op `McpSafeAssetSave` (`AssetUtils.cpp:214-226` — `MarkPackageDirty`+`AssetCreated`, no disk write) and reported a hardcoded `existsAfter:true` (`AddAssetVerification`, `AssetUtils.cpp:1091-1100`) with no `saved`/`pendingFlush`/`sizeBytes`. FIX: replaced the `McpSafeAssetSave(StaticMesh)` call in all three verbs with `SaveAssetToDiskReportingPresence(StaticMesh, /*bForce=*/true, &pkg, &sizeBytes)` (forced `SaveLoadedAsset` + `IFileManager::FileSize` disk probe gated by `ShouldTreatAssetSaveAsSuccess`, `AssetUtils.cpp:418-449`) and added `AddAssetSaveReport(Result, /*saveRequested=*/true, bSavedToDisk)` (`AssetUtils.cpp:451-466`) + a `sizeBytes` field beside the existing `AddAssetVerification` — the exact established pattern the accepted convert `#3-fix` shipped in `MeshOpsHandler.cpp:436-479`. A dirty-only outcome now reports `saved:false`+`pendingFlush:true` instead of a bare hardcoded `existsAfter:true`. Files: `Plugins/PinWright/Source/PinWright/Private/Handlers/Geometry/LODCollisionHandler.cpp`. Regression test: `PinWright.geometry.generate_lods.SavesToDisk` (`Plugins/PinWright/Source/PinWright/Private/Tests/Geometry/TestGenerateLodsSavesToDisk.cpp`) — builds an in-code fixture (`geometry.create_box` -> `geometry.convert_to_static_mesh` to an on-disk StaticMesh), records the post-convert on-disk size, then drives `geometry.generate_lods{lodCount:4}` through the real dispatcher and asserts the response carries `saved:true`+positive `sizeBytes`+no `pendingFlush` AND the `.uasset` on disk GREW versus baseline (the +LOD-chain bytes the live replay measured, 764033->773193). Reverting to `McpSafeAssetSave` writes nothing, so both the `saved:true` and grew-on-disk assertions fail. No external/Lyra content loaded. Sibling verbs (`set_lod_settings`/`set_lod_screen_sizes`) got the identical treatment. Compiled clean (`EAContentExamples57Editor` Win64 Development).
