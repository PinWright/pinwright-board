---
id: B-nanite-rebuild-mesh-no-disk-write
title: "asset.nanite_rebuild_mesh reports naniteEnabled:true and never writes the .uasset — the flag is dirty-in-memory only, the response carries no save/persistence field at all, and a read-back of nanite_settings.enabled cannot tell the two states apart"
status: OPEN
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

## History
- `#1-filed` `OPEN` reporter — Hit while enabling Nanite across the 18-mesh FPS environment kit on UE 5.8 / EAContentExamples58. Called `asset.nanite_rebuild_mesh {enableNanite:true}` on four meshes (`SM_ENV_WallPanel`, `SM_ENV_Slab`, `SM_ENV_GroundTile`, `SM_ENV_Container20`), each returning the full settings block quoted above with `naniteEnabled:true` and no persistence field; did the remaining 13 through `python.execute` with an explicit `mark_package_dirty` + `save_asset(only_if_is_dirty=False)`. A directory `ls` immediately afterwards showed the 13 python-path files at 23:23:4x and exactly the 4 verb-path files still at their previous `model.compile` timestamps. Re-probing those four gave `dirtyBefore: true` for all of them — so the mutation and the dirty flag both happened and only the write was missing — and a forced save moved all four to 23:24:3x. Also recording the second-order trap, because it is the part that would bite a careful caller: my sweep guarded the save behind `if not enabled`, which read `true` off the in-memory object and skipped the repair, so the sweep *also* reported those four as done. Dedup: grepped the board for `nanite`, `nanite_rebuild_mesh` and `no-disk-write`; the three neighbouring tickets are analysed as distinct above. Workaround in force for my stream: never use this verb's own report as evidence — follow every call with `asset.save {force:true}` or the explicit Python dirty+save, and confirm by `.uasset` mtime.
