---
id: B-create-level-saved-true-no-umap
title: "level.structure.create_level reports saved:true/existsAfter:true but writes no .umap to disk, stranding an orphaned in-memory world"
status: IN-REVIEW
severity: High
category: bug
tags: [level, create-level, save, silent-failure, false-success, orphan, mcp-safe-level-save]
encounters: 2
lastSeen: 2026-07-04T16:17:14.0325862+03:00
---

# level.structure.create_level claims it saved a level it never wrote to disk

`level.structure.create_level` returns a success payload with
`{saved:true, existsAfter:true, assetClass:"World"}`, but **no `.umap` file is
written to disk**. The world exists only in memory + the asset registry, so the
caller is left with an unrecoverable orphan:

- A retry with the same name → `[LEVEL_ALREADY_EXISTS] Level already exists in
  memory: <path>` (the in-memory `UWorld` blocks recreation —
  `LevelStructureHandler.cpp:194-203`).
- `level.load` / `editor.open_level` on the reported path → `[FILE_NOT_FOUND]`
  (there is nothing on disk to load).
- `editor.save_all` sees `totalDirty:0` (the package's dirty flag was already
  cleared by the bogus "save"), so the in-memory world cannot be flushed either.

Net: the map can never become the active world, and every downstream operation
that needs it as a target is impossible. The `saved:true`/`existsAfter:true`
response **falsely confirms** an asset that does not exist on disk — a classic
silent success-with-no-effect.

**This is not World-Partition-specific.** The flat (`bCreateWorldPartition:false`)
path is equally broken — it also returns `saved:true` and leaves no `.umap`.
(The separate `worldPartitionEnabled:false` field *is* honestly reported and
carries a `worldPartitionNote`, so WP-not-enabled is a disclosed limitation, not
the bug here — the bug is the false `saved:true` plus the orphan.)

## Root cause

`create_level` saves via `McpSafeLevelSave(NewWorld->PersistentLevel, FullPath, 5)`
(`LevelStructureHandler.cpp:256`), and the response's `saved` field is set to
`bSave && bSaveSucceeded` (`:282`). `McpSafeLevelSave`
(`Utils/AssetUtils.cpp:303-399`) calls `FEditorFileUtils::SaveLevel`, then gates
success on `ShouldTreatLevelSaveAsSuccess(bSaveReportedSuccess, bFileExistsOnDisk,
bPackageExists, bAssetExists, bPackageClean)` (`AssetUtils.cpp:288-301`):

```cpp
if (!bSaveReportedSuccess) return false;
return bFileExistsOnDisk || bPackageExists || bAssetExists || bPackageClean;
```

The success policy is an **OR over four signals**, three of which are satisfied
purely by in-memory / registry state. When `FEditorFileUtils::SaveLevel` returns
`true` and clears the package dirty flag *without producing a `.umap`* (the
observed UE 5.7 behaviour for a freshly `CreateWorld`'d inactive world whose
package was just `CreatePackage`'d), `bPackageClean` (and/or `bAssetExists` via
the registry) is `true`, so the disk file never landing on disk
(`bFileExistsOnDisk == false`) is masked. `McpSafeLevelSave` returns `true`,
`create_level` reports `saved:true`, and the file is simply absent. There is no
JSON-level signal of the discrepancy. (The handler even runs
`AssetRegistry.ScanFilesSynchronous` on the would-be filename at `:262-268`,
which is why `level.list` subsequently lists the path despite no disk file.)

This exact weakness was already flagged internally as a suggestion — "Update
`McpSafeLevelSave` success criteria to handle package/mount workflows where
`SaveLevel` succeeds but direct `.umap` file probe path is not valid" (codex-cycle
0003 executor log) — but was never turned into a fix or a board ticket.

## Why it matters

This is the very first step of an extremely common task ("create a new map and
save it"). Because the response says `saved:true`/`existsAfter:true`, an agent
reasonably treats the map as persisted and moves on — then every attempt to make
it the active world (`level.load`, `editor.open_level`) fails with FILE_NOT_FOUND
while recreation is blocked by LEVEL_ALREADY_EXISTS. The orphan also leaks an
in-memory `UWorld` for the rest of the editor session.

## Fix scope (corrected)

**Do NOT change the shared `ShouldTreatLevelSaveAsSuccess` predicate.** Its
OR-policy is *deliberate* and is locked by intentional unit tests
(`Tests/Core/TestLevelSaveLoadUtils.cpp:84-108`:
`FLevelSavePolicyAcceptsPackageOrAssetTest` asserts
`ShouldTreatLevelSaveAsSuccess(true,false,true,false,false)`→true and
`(true,false,false,true,false)`→true; `FLevelSavePolicyAcceptsCleanPackageTest`
asserts `(true,false,false,false,true)`→true). Those cover package/mount
workflows where the direct `.umap` file probe is not valid. Requiring
`bFileExistsOnDisk` *inside the predicate* (the original Fix option 1) would
break those tests and that workflow. The defect is not the predicate; it is that
`create_level` applies the lenient package/clean fallback to a **create-to-a-new
`/Game/` path** flow where the `.umap` landing on disk is the only honest
success signal.

The fix belongs in the `create_level` handler (`LevelStructureHandler.cpp`),
which already proved the destination has no prior on-disk package
(`FPackageName::DoesPackageExist(FullPath)` returned false at `:206`). After the
save:

1. **Verify disk presence for this create-to-new-path flow.** Convert `FullPath`
   to a `.umap` filename and probe `IFileManager::Get().FileExists`. If the file
   is absent, treat the save as failed for this flow (the cleared dirty flag /
   registry entry are not honest persistence signals for a brand-new path).
2. **Fail loud.** When the disk file is absent, drive the existing
   `SAVE_VERIFICATION_FAILED` branch (`LevelStructureHandler.cpp:288-292`) so the
   response is no longer a false `saved:true`/`existsAfter:true`.
3. **Don't orphan on failure.** Before returning the error, destroy / GC the
   in-memory `UWorld` and rename the package out of the way so a retry with the
   same name isn't permanently blocked by `LEVEL_ALREADY_EXISTS` (`:194-203`).

This keeps the shared predicate (and its package/mount tests) intact while making
disk presence load-bearing exactly where the workflow guarantees it must hold.

## Repro (replayed via mcp__editor-automation__call)

1. `level.structure.create_level {levelName:"FuzzReplay_WP_A", levelPath:"/Game/Maps", bCreateWorldPartition:true, save:true}`
   → `{assetPath:"/Game/Maps/FuzzReplay_WP_A", existsAfter:true, assetClass:"World",
   worldPartitionEnabled:false, worldPartitionRequested:true, saved:true,
   worldPartitionNote:"World Partition must be enabled via editor UI or project settings for new levels"}`.
   On-disk check: `Content/Maps/FuzzReplay_WP_A.umap` does **not** exist
   (`Content/Maps/` contains only `ExampleProjectWelcome.umap` + subfolders).
2. `level.structure.create_level {levelName:"FuzzReplay_Flat_B", ..., bCreateWorldPartition:false, save:true}`
   → `{existsAfter:true, assetClass:"World", saved:true}`. On-disk:
   `Content/Maps/FuzzReplay_Flat_B.umap` does **not** exist either → the false
   `saved:true` is not WP-specific.
3. Re-issue step 1's call (same name) →
   `[LEVEL_ALREADY_EXISTS] Level already exists in memory: /Game/Maps/FuzzReplay_WP_A.`
4. `level.load {levelPath:"/Game/Maps/FuzzReplay_WP_A"}` →
   `[FILE_NOT_FOUND] Level file not found: /Game/Maps/FuzzReplay_WP_A`.
   The map is registered + orphaned in memory, blocks recreation, but cannot be
   loaded (nothing on disk) or flushed (`editor.save_all` → `savedCount:0,
   totalDirty:0`).

## History
- `#5-additional-cold-load` `IN-REVIEW` reporter — Additional cold-load confirmation from a lighting-sandbox setup task ("create /Game/Maps/LightingSandbox with a directional + point light, save it, make a backup, delete the backup"). A real editor cold-restart (editor.quit discard + relaunch; gateway came back up clean on auto-derived port 24966, no crash) cold-opened three warm-"saved" maps — `/Game/Maps/LightingSandbox`, `/Game/Maps/PW_DiskTest_LS`, `/Game/Maps/LS_Tmp` — and ALL THREE returned `[ASSET_NOT_FOUND]`; no `.umap` exists on disk for any of them (`Content/Maps/` holds only `ExampleProjectWelcome.umap`, which cold-opens cleanly). Same persistence-loss consequence as `#4`, but this session spanned MULTIPLE creation verbs at once: `level.structure.create_level {save:true}` (this ticket — `[SAVE_VERIFICATION_FAILED] Level created but no .umap was written to disk` on both LightingSandbox and PW_DiskTest_LS, honest per the `#2` fix), `level.create` (LightingSandbox/LS_Tmp — see `B-level-create-makes-wp-map`), `level.save`/`level.save_as` (honest SAVE_VERIFICATION_FAILED per `B-level-save-saved-true-in-memory-no-umap`), and `lighting.create_lighting_enabled_level` (which — unlike the now-hardened level.* verbs — STILL returns a FALSE `success:true`/`existsAfter:true` on the no-op; filed separately as `B-lighting-create-level-false-success-no-umap`). Not a new corruption vector (editor stayed healthy through the restart and every open, no malformed `.umap` written) — it is the downstream `load_failed` consequence of the still-unfixed in-memory-world disk write: as long as no MCP route persists a freshly-created level's `.umap`, a cold load finds nothing. Still reproduces at HEAD.
- `#4-additional-cold-load-confirms-no-persist` `OPEN` reporter — Additional evidence — **a real editor cold-restart confirms the persistence defect downstream**, not just an on-disk probe. An open-world-WP setup task ("create WP map `OpenWorldSandbox` under /Game/Maps, 3 data layers, spawn+assign actors, save") hit this defect at every save path: `level.structure.create_level {OpenWorldSandbox, /Game/Maps, bCreateWorldPartition:true, save:true}` → `[SAVE_VERIFICATION_FAILED] Level created but no .umap was written to disk: /Game/Maps/OpenWorldSandbox` (×3 across retries); the `save:false` workaround built the active in-memory WP world (`worldPartitionEnabled:true`, `activeWorld:true`), the agent created all 3 data layers + spawned/assigned actors in memory, then `level.save` and `level.save_as` BOTH returned `SAVE_VERIFICATION_FAILED` (`saved:false` — honestly; the `B-level-save-saved-true-in-memory-no-umap` fix correctly reports `false` now, no false `saved:true`). The orchestrator then **cold-restarted the editor** (`editor.quit` discard, relaunch headless) and re-opened all four supposedly-created assets via `editor.open_asset`: `/Game/Maps/OpenWorldSandbox`, `/Game/DataLayers/Foliage`, `/Game/DataLayers/Enemies`, `/Game/DataLayers/BlockingVolumes` ALL returned `[ASSET_NOT_FOUND] Asset not found`; filesystem confirmed no `Content/Maps/OpenWorldSandbox.umap` and **no `Content/DataLayers/` folder at all**. This is NOT a new asset-corruption vector (the editor stayed up, never crashed, no malformed `.umap` was written) — it is the *consequence* of this ticket's persistence defect: because the WP world could never be written to disk, neither the `.umap` nor the data-layer packages (which only persist with their host world's save) ever landed, so a real cold load can't find them. Replayed on the live IN-REVIEW build and still reproduces verbatim: `create_level {OracleColdReplay_WP1, /Game/Maps, bCreateWorldPartition:true, save:true}` → `[SAVE_VERIFICATION_FAILED] ... /Game/Maps/OracleColdReplay_WP1`; `create_level {OracleColdReplay_WP2, ..., save:false}` → active WP world (`worldPartitionEnabled:true`, `activeWorld:true`); `level.save_as {savePath:/Game/Maps/OracleColdReplay_WP2}` (async job) → `system.job_status` → `result:{saved:false}`, `error:SAVE_VERIFICATION_FAILED`, `persistenceNote:"Save reported success but no .umap was written to disk — the active world could not be persisted to the requested path."`; on-disk re-check found no `OracleColdReplay*.umap` and still no `Content/DataLayers/` folder. The `#3` fix is honest but the underlying disk-write is still unfixed: as long as no MCP path persists an in-memory active WP world, the whole "create WP map → data layers → save" workflow remains a hard dead-end whose only on-disk trace is *nothing*. (Seed `level.structure.create_data_layer`: the data-layer verbs themselves worked in-memory and the structure readback registered all three correctly; the friction lands entirely on `create_level`/`level.save`/`level.save_as` persistence.)
- `#3-additional-save-still-broken-both-paths` `OPEN` reporter — Additional evidence (replayed against the live editor on the IN-REVIEW build): the `#2` fix correctly killed the *false* `saved:true` (the report is now honest), but the **underlying persistence defect is unfixed and now hard-blocks the whole create-and-save workflow** — and it is **NOT World-Partition-specific**. `level.structure.create_level {levelName:"OracleWPReplay_Z1", levelPath:"/Game/Maps", bCreateWorldPartition:true, save:true}` → `[SAVE_VERIFICATION_FAILED] Level created but no .umap was written to disk: /Game/Maps/OracleWPReplay_Z1`; **`{levelName:"OracleFlatReplay_Z3", bCreateWorldPartition:false, save:true}` → the SAME `[SAVE_VERIFICATION_FAILED] ... /Game/Maps/OracleFlatReplay_Z3`** (flat path fails identically — so the missing `.umap` is the save itself, not WP). On-disk verify: `Content/Maps/` still holds only `ExampleProjectWelcome.umap`; a recursive search for `Oracle*`/`Replay*` under `Content/` found nothing — no file landed at the resolved path nor anywhere else. The WP `save:false` control DID succeed and now reports `worldPartitionEnabled:true` (so the `F-enable-world-partition-impossible` IVS fix genuinely attaches a `UWorldPartition`); only the `save:true` disk-write is broken. Root cause sits one layer below the new verifier: `McpSafeLevelSave` (`LevelStructureHandler.cpp:281`) calls `FEditorFileUtils::SaveLevel(Level, *PackagePath)` (`Utils/AssetUtils.cpp:443`) which **reports success and clears the dirty flag but never writes the `.umap`** for a freshly `CreateWorld`'d inactive world that was never made the active editor world — so `ShouldTreatCreateLevelSaveAsSuccess(true, /*bFileOnDisk=*/false)` correctly returns false and the handler tears the world down. Net: with `save` defaulting to `true`, `create_level` can no longer persist *any* new level (WP or flat) — the very first step of "create a new map and save it" is a hard dead-end. The `#2` fix is honest but incomplete: making the save actually land the `.umap` (e.g. set the created world active before `SaveLevel`, or save via the active-world `FEditorFileUtils::SaveMap` rename path that `level.create` uses — see `E-level-create-active-world-mismatch` `#2`) is still required. (Seed was `level.structure.create_minimap_volume`, which was unreachable because no persisted active WP world could be produced — the friction lands entirely on `create_level`.)
- `#2-reword-and-fix` `IN-REVIEW` developer — REWORD then fix. Adversarial lens (verified) showed the original Fix option 1 (require `bFileExistsOnDisk` inside `ShouldTreatLevelSaveAsSuccess`) would break the intentional package/mount-workflow tests at `Tests/Core/TestLevelSaveLoadUtils.cpp:84-108`, so I rewrote the **Fix scope** section to leave that shared predicate (and its tests) untouched and retarget the fix to the `create_level` handler's create-to-new-`/Game/`-path flow. Implementation: (1) added a stricter `ShouldTreatCreateLevelSaveAsSuccess(bSaveReportedSuccess, bFileExistsOnDisk)` predicate in `Utils/AssetUtils.cpp`/`.h` (`saved` requires the `.umap` to actually land on disk — honest because `create_level` already proved `DoesPackageExist==false` for the destination at `:206`); (2) `LevelStructureHandler.cpp` `create_level` now probes the `.umap` via `IFileManager::FileExists` after `McpSafeLevelSave` and runs the result through the new predicate, so a save that reports success with no disk file now drives the existing `SAVE_VERIFICATION_FAILED` branch instead of a false `saved:true`/`existsAfter:true`; (3) on that failure it tears down the orphaned in-memory `UWorld` (`DestroyWorld`) and renames/garbages the package out of the way so a retry isn't permanently blocked by `LEVEL_ALREADY_EXISTS`. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Level/LevelStructureHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Utils/AssetUtils.cpp`, `Source/EditorAutomationRpcGateway/Private/Utils/AssetUtils.h`. Test: added `FCreateLevelSavePolicyRequiresDiskFileTest` in `Source/EditorAutomationRpcGateway/Private/Tests/Core/TestLevelSaveLoadUtils.cpp` exercising the production `ShouldTreatCreateLevelSaveAsSuccess` — asserts the general OR-policy still masks a missing file (clean-package fallback) while the create policy FAILS on `(true,false)` and passes only on `(true,true)`; reverting the handler to the lenient OR-policy result regresses this test. Did not compile/run (later phase).
- `#1-initial-repro` `OPEN` reporter — Replayed `level.structure.create_level` for both `bCreateWorldPartition:true` (`FuzzReplay_WP_A`) and `false` (`FuzzReplay_Flat_B`); both returned `{saved:true, existsAfter:true, assetClass:"World"}` while no `.umap` was written to `Content/Maps/` (verified on disk — only `ExampleProjectWelcome.umap` present). Re-creating the same name → `[LEVEL_ALREADY_EXISTS]`; `level.load` on the reported path → `[FILE_NOT_FOUND]`. Root cause in `Utils/AssetUtils.cpp:288-301` (`ShouldTreatLevelSaveAsSuccess` OR-policy lets `bPackageClean`/`bAssetExists` mask a missing disk file) called from `LevelStructureHandler.cpp:256`; `saved` is reported from this masked result at `:282`. Same `McpSafeLevelSave` weakness an internal codex-cycle executor log already flagged as a suggestion but never filed. Dedup: distinct from `B-level-save-no-completion-signal`/`B-level-save-as-no-completion-signal` (DONE; those are async-completion-signal issues on `level.save`/`level.save_as`, not a false `saved:true` with no disk file) and from `B-configure-world-partition-silent-noop` (OPEN; that's `performance.configure_world_partition` CVar echoing). No existing board entry for this defect (ripgrep + qmd clean).
