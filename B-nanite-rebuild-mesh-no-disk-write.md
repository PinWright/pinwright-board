---
id: B-nanite-rebuild-mesh-no-disk-write
title: "asset.nanite_rebuild_mesh reports naniteEnabled:true and never writes the .uasset — the flag is dirty-in-memory only, the response carries no save/persistence field at all, and a read-back of nanite_settings.enabled cannot tell the two states apart"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, nanite_rebuild_mesh, nanite, static-mesh, save, no-disk-write, silent-failure, false-success, persistence, mtime, AssetWorkflowHandler]
encounters: 1
lastSeen: 2026-09-02T20:24:00+00:00
---

# `asset.nanite_rebuild_mesh` enables Nanite in memory and leaves the asset on disk untouched

## Symptom

Four calls, one per mesh, each returning a full settings block and no error:

```jsonc
call("asset.nanite_rebuild_mesh", {meshPath: "/Game/FPS/Env/Meshes/SM_ENV_WallPanel",
                                   enableNanite: true})
-> { "meshPath": "/Game/FPS/Env/Meshes/SM_ENV_WallPanel", "meshName": "SM_ENV_WallPanel",
     "naniteEnabled": true, "shapePreservation": "none", "preserveArea": false,
     "positionPrecision": -2147483648, "positionPrecisionAuto": true,
     "trianglePercent": 100, "fallbackPercent": 100,
     "generateFallback": "PlatformDefault", "foliageOverOcclusionBias": 0 }
```

There is **no `saved`, `saveRequested`, `saveState`, `markedForSave` or `pendingFlush` field
anywhere in that payload** — nothing to read, so nothing to be suspicious of. The `.uasset` was
not written.

## The measurement that separates it

I enabled Nanite on 17 kit meshes in the same minute: 4 through this verb, 13 through
`python.execute` doing `nanite_settings.enabled = True` +
`PinWrightPackageLibrary.mark_package_dirty` + `EditorAssetLibrary.save_asset(only_if_is_dirty=False)`.
Then `ls` on the directory:

```
23:23:46  SM_ENV_Barrier_Jersey.uasset     <- python path, written
23:23:47  SM_ENV_Barrel.uasset             <- python path, written
23:23:48  SM_ENV_Column_IBeam.uasset       <- python path, written
23:23:48  SM_ENV_Crate.uasset              <- python path, written
   ... 9 more, all 23:23:4x ...
23:08:07  SM_ENV_WallPanel.uasset          <- nanite_rebuild_mesh, STALE (last model.compile)
22:26:32  SM_ENV_Slab.uasset               <- nanite_rebuild_mesh, STALE
22:26:33  SM_ENV_GroundTile.uasset         <- nanite_rebuild_mesh, STALE
22:44:09  SM_ENV_Container20.uasset        <- nanite_rebuild_mesh, STALE
```

Exactly the four meshes this verb touched are the four whose file did not move. Their stale
timestamps are each that mesh's last `model.compile`, minutes to an hour earlier.

Then, forcing the save by hand on those same four:

```
SM_ENV_WallPanel   naniteInMemory=True  dirtyBefore=True  dirtyAfterSave=False
SM_ENV_Slab        naniteInMemory=True  dirtyBefore=True  dirtyAfterSave=False
SM_ENV_GroundTile  naniteInMemory=True  dirtyBefore=True  dirtyAfterSave=False
SM_ENV_Container20 naniteInMemory=True  dirtyBefore=True  dirtyAfterSave=False
```

`dirtyBefore: true` on all four is the proof: the verb **did** mutate the object and **did** dirty
the package, and then stopped. After the manual save the four moved to 23:24:3x on disk.

## Why the obvious guard does not catch it

`nanite_settings.enabled` reads back `true` either way, because it is reading the in-memory
object — the exact trap `CLAUDE.md` § "Verify a write against disk" names. Worse, it makes a
**reasonable idempotent script skip the repair**: my sweep was written as

```python
if not ns.get_editor_property("enabled"):
    ns.set_editor_property("enabled", True)
    ... mark dirty, save ...
back = m.get_editor_property("nanite_settings").get_editor_property("enabled")
```

so for those four the guard saw `enabled == True`, skipped the save, and reported them in the
`NANITE ON` list. Both the verb and the follow-up sweep reported success; only `ls` disagreed.
Any caller who mixes this verb with a save-on-change script inherits the same hole.

## What should happen

1. **Save, or say that it did not.** Either flush the package (the sibling `asset.*` writers do)
   or add the persistence block the rest of the plugin publishes — `saveRequested` / `saved` /
   `saveState` per `safe-mutation-save`, so `pendingFlush` tells the caller to call `asset.save`.
   A response with no persistence field at all is the part that makes this invisible.
2. **Accept a `save` parameter**, as `material.authoring.set_material_instance_parameters` and
   `material.compile_mgir` already do, so the common case is one call.
3. Failing both, the wiki page must say the write is in-memory only and name `asset.save` as the
   required follow-up. It currently says "every reported setting is read back off the asset after
   the write", which reads as a durability claim and is the sentence that convinced me the four
   were done.

## Related

- `B-geometry-convert-static-mesh-no-disk-write` (IN-REVIEW, Critical) — same failure shape
  (`created:true` with no flush) on a different verb, `geometry.convert_to_static_mesh`
  (`MeshOpsHandler.cpp:366-446`). This one is `asset.nanite_rebuild_mesh`, whose own registration
  is at `Handlers/Asset/AssetWorkflowHandler.cpp:1249` per
  `B-render-nanite-rebuild-mesh-no-completion-signal` `#5`; that ticket notes the `asset.*` verb is
  "a separate, richer registration", not an alias of the `render.*` one, so a fix there does not
  reach here.
- `B-render-nanite-rebuild-mesh-no-completion-signal` (DONE) — the *async completion* signal for
  the rebuild job, a different missing signal on the neighbouring verb. Persistence was never in
  scope there.
- `B-nanite-rebuild-preserve-area-boolean-hides-voxelize` — a parameter-surface issue on the same
  verb, unrelated to persistence.

severity rationale: impact=silent loss of an asset setting with no field in the response that could
warn, plus it defeats the standard read-back-and-skip idempotent sweep x reach=every caller
enabling Nanite through the typed verb, which is the documented route -> High

## Fix

The asset handler stopped after changing Nanite settings and dirtying the package, while the render handler stopped after its async build and hardcoded success. Neither handler finished pending StaticMesh compilation, saved the loaded asset, probed the package file, or exposed the standard persistence verdict. Both handlers now accept `save` (default `true`), retain their separate synchronous/settings-rich and async/job shapes, and use `SaveAssetToDiskReportingPresence(..., /*bForce=*/true)` plus `AddAssetSaveSizeReport` and `AddAssetSaveReport`. The asset path finishes compilation synchronously; the render job retains the mesh and polls compilation through a bounded ticker, then performs its final drain, optional save/probe, and terminal result. Render cancellation and timeout remove the ticker, route cancellation through `FAssetCompilingManager`, synchronously drain any work the engine could not cancel, release retained state, and do not save. An invalid retained mesh now resolves the ticket as failed with `INVALID_ASSET` and `saveState:"failed"` instead of silently stopping its ticker. With `save:false`, both handlers report the real package and any pre-existing `.uasset` size as stale while leaving that disk revision untouched and the in-memory package dirty.

Files changed:

- `unreal-fpv-dev/Plugins/PinWright/.codex/plans/mcp-sprint-2026-09-03-nanite-save.md` — implementation plan and bounded verification contract.
- `unreal-fpv-dev/Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/AssetWorkflowHandler.cpp` — synchronous rebuild, compilation wait, optional verified save, and persistence report.
- `unreal-fpv-dev/Plugins/PinWright/Source/PinWright/Private/Handlers/Render/RenderHandler.cpp` — async rebuild, bounded compilation ticker with cancellation/timeout cleanup, optional verified save, and terminal persistence report.
- `unreal-fpv-dev/Plugins/PinWright/Source/PinWright/Private/Tests/Assets/NaniteRebuildSaveTestUtils.h` — shared mesh fixture with compilation-safe cleanup, baseline disk snapshot, JSON readers, and persistence assertions.
- `unreal-fpv-dev/Plugins/PinWright/Source/PinWright/Private/Tests/Assets/TestNaniteRebuildMeshSavesToDisk.cpp` — asset-handler disk and opt-out regression.
- `unreal-fpv-dev/Plugins/PinWright/Source/PinWright/Private/Tests/EditorOps/TestRenderNaniteRebuildMeshSavesToDisk.cpp` — render-job disk and opt-out regression with asset-compilation pumping and cancellation-safe watchdog cleanup.
- `unreal-fpv-dev/Plugins/PinWright/docs/wiki-src/asset.md` — asset verb save and `save:false` stale-size contract.
- `unreal-fpv-dev/Plugins/PinWright/docs/wiki-src/render.md` — bounded async compilation and terminal save contract.
- `unreal-fpv-dev/Plugins/PinWright/docs/wiki-src/system.md` — ticket-pattern and cancellation support update.
- `.pinwright-board/B-nanite-rebuild-mesh-no-disk-write.md` — status, fix record, and history.

Regression tests: `PinWright.asset.nanite_rebuild_mesh.SavesToDisk` and `PinWright.render.nanite_rebuild_mesh.SavesToDisk`. Each invokes the registered production handler, proves the default writes a non-empty `.uasset` and cleans the package, and proves `save:false` leaves a pre-existing `.uasset` byte-identical while reporting its real package and stale size and leaving the package dirty. The render test advances compilation through `PinWright::AssetCompile::AdvanceOnGameThread`, polls the real job registry, asserts persistence only on the terminal result, invokes `system.job_cancel` for its cancellation contract, and source-checks the otherwise non-injectable retained-mesh invalidation and manager-owned cancellation/drain guards. Its watchdog cancels the real ticket and drains the mesh before fixture cleanup. Both tests share one header-only fixture/assertion helper whose destructor conditionally drains active mesh compilation before unrooting and deleting the asset.

### Remaining

Compilation, automation, and runtime behavior remain unverified. The invalid-retained-mesh guard is covered structurally because the production `TStrongObjectPtr` deliberately prevents normal GC invalidation; no test-only production hook was added. The cancellation regression covers the public job response and terminal ticket state, while active in-flight compiler cancellation remains source-verified against UE 5.8 rather than runtime-proven.

Deliberate non-changes: the unlike-shaped asset and render handler bodies were not merged; `Utils/AssetUtils.*`, the unrelated asset-handler hunks near the concurrent edit, and `B-geometry-convert-static-mesh-no-disk-write.md` were not changed. No Unreal process, MCP call, build, compile, or test run was performed in this implementation pass.

## History
- `#1-filed` `OPEN` reporter — Hit while enabling Nanite across the 18-mesh FPS environment kit on UE 5.8 / EAContentExamples58. Called `asset.nanite_rebuild_mesh {enableNanite:true}` on four meshes (`SM_ENV_WallPanel`, `SM_ENV_Slab`, `SM_ENV_GroundTile`, `SM_ENV_Container20`), each returning the full settings block quoted above with `naniteEnabled:true` and no persistence field; did the remaining 13 through `python.execute` with an explicit `mark_package_dirty` + `save_asset(only_if_is_dirty=False)`. A directory `ls` immediately afterwards showed the 13 python-path files at 23:23:4x and exactly the 4 verb-path files still at their previous `model.compile` timestamps. Re-probing those four gave `dirtyBefore: true` for all of them — so the mutation and the dirty flag both happened and only the write was missing — and a forced save moved all four to 23:24:3x. Also recording the second-order trap, because it is the part that would bite a careful caller: my sweep guarded the save behind `if not enabled`, which read `true` off the in-memory object and skipped the repair, so the sweep *also* reported those four as done. Dedup: grepped the board for `nanite`, `nanite_rebuild_mesh` and `no-disk-write`; the three neighbouring tickets are analysed as distinct above. Workaround in force for my stream: never use this verb's own report as evidence — follow every call with `asset.save {force:true}` or the explicit Python dirty+save, and confirm by `.uasset` mtime.
- `#2-nanite-save-report` `IN-REVIEW` developer — Changed asset.nanite_rebuild_mesh and render.nanite_rebuild_mesh to accept save:true by default, force-save through SaveAssetToDiskReportingPresence, and report saveRequested/saved/pendingFlush/saveState; added the asset and render disk-persistence regressions.
- `#3-nanite-job-cleanup` `IN-REVIEW` developer — Made the render Nanite job resolve invalid retained meshes as explicit failures, routed cancellation through compiler-manager bookkeeping with a guaranteed drain before release, and extended the existing render regression for shared compile pumping and cancellation/terminal guards.
