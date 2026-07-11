---
id: B-pose-search-create-save-no-disk-write
title: "pose_search.create_schema / create_database / add_database_animation (save:true) report saved:true + existsAfter:true but McpSafeAssetSave only marks dirty — nothing reaches disk, so the Motion Matching assets vanish on cold restart"
status: IN-REVIEW
severity: Critical
category: bug
tags: [pose-search, motion-matching, create-schema, create-database, add-database-animation, save, mcp-safe-asset-save, no-disk-write, cold-load, persistence, silent-failure, false-success]
encounters: 1
lastSeen: 2026-07-11T03:16:36.4063281+03:00
claimedBy: fuzz2
claimedAt: 2026-07-11T03:53:02.9842612+03:00
---

# pose_search.create_* claim success but write nothing to disk — the schema + database are lost on restart

The `pose_search` authoring create/mutate handlers — `create_schema`,
`create_database`, and `add_database_animation` — take a `save` param defaulting
to `true`, return an `AddAssetVerification` block with `existsAfter:true` plus an
explicit `saved:true`, and so make a caller reasonably believe the new
`UPoseSearchSchema` / `UPoseSearchDatabase` `.uasset` is on disk. It is not. With
`save:true` the handler only **marks the package dirty + notifies the asset
registry**; the file is never written. The asset exists only in memory + the
registry (hence `existsAfter:true`, and the in-session `asset.dump` / `asset.get`
readbacks succeed), but after the editor closes (or a fuzz `git reset --hard`)
**the asset is gone**, with no error ever surfaced.

This is the **Pose Search / Motion Matching analog** of the shared-helper
no-disk-write family — `B-metasound-create-save-no-disk-write` and
`B-niagara-save-no-disk-write` (both on this same `McpSafeAssetSave`),
`B-audio-create-save-no-disk-write` (`SaveAudioAsset`), and
`B-material-authoring-save-no-disk-write` (`SaveMaterialAsset`). The
`pose_search.*` namespace was newly added by `F-pose-search-database-authoring`
and routes its save through the shared mark-dirty-only `McpSafeAssetSave`, so it
inherits the exact same silent persistence loss; no pose_search save ticket
existed, so this fills the gap.

## Cold-load-confirmed asset loss (a real editor restart)

A REALISM-mode Motion Matching task built a reusable Pose Search setup for the
Manny character: `pose_search.create_schema` (a `UPoseSearchSchema` on
`SK_Mannequin`, 3 Position channels root/foot_l/foot_r) at
`/Game/Animation/MotionMatching/PS_Manny_Locomotion_Schema`, then
`pose_search.create_database` bound to that schema with 4 Manny locomotion clips
at `/Game/Animation/MotionMatching/DB_Manny_Locomotion`. Both create calls
returned success with `saved:true` + `existsAfter:true`, and the in-session
`asset.dump` readback confirmed the schema (skeleton + 3 Position channels) and
the database (schema ref + 4 sequence entries) as "persisted".

A `CorruptionCheck` cold restart (editor quit with discard, relaunched headless,
gateway came back up cold) then found **both** newly-created assets ENTIRELY
ABSENT on disk:

```
editor.open_asset /Game/Animation/MotionMatching/PS_Manny_Locomotion_Schema -> [ASSET_NOT_FOUND] Asset not found
asset.exists   /Game/Animation/MotionMatching/PS_Manny_Locomotion_Schema     -> exists:false
asset.validate /Game/Animation/MotionMatching/PS_Manny_Locomotion_Schema     -> [ASSET_NOT_FOUND] Asset not found

editor.open_asset /Game/Animation/MotionMatching/DB_Manny_Locomotion -> [ASSET_NOT_FOUND] Asset not found
asset.exists   /Game/Animation/MotionMatching/DB_Manny_Locomotion     -> exists:false
asset.validate /Game/Animation/MotionMatching/DB_Manny_Locomotion     -> [ASSET_NOT_FOUND] Asset not found
```

The entire `Content/Animation/` folder does not exist on disk — no
`*MotionMatching*` / `*Manny_Locomotion*` file anywhere under `Content` — even
though the warm pre-restart session's `asset.get` returned both from the
in-memory package. The editor stayed fully healthy throughout (each probe
returned a clean structured `ASSET_NOT_FOUND`) — not an editor crash. The
in-session `asset.dump` "confirmation" read the registry/in-memory package, not
disk, which is exactly why it masked the loss. (3 of the 5 assets the task
touched — `SK_Mannequin`, `MM_Rifle_Jog_Fwd`, `MM_Run_Fwd` — were pre-existing
on-disk clips and cold-loaded intact; only the two newly-created pose_search
assets are the persistence losses.)

## Root cause (verified in source)

`Source/PinWright/Private/Handlers/PoseSearch/PoseSearchHandler.cpp` routes every
create/mutate `save:true` through the shared mark-dirty-only helper:

- `HandleCreateSchema` — `:486` `McpSafeAssetSave(Schema);` (under `if (Ctx.GetBool(TEXT("save"), true))`).
- `HandleCreateDatabase` — `:594` `FinishPoseSearchAsset(Database, Ctx.GetBool(TEXT("save"), true));` which at `:159` calls `McpSafeAssetSave(Asset);`.
- `HandleAddDatabaseAnimation` — `:639` `FinishPoseSearchAsset(Database, Ctx.GetBool(TEXT("save"), true));` -> `:159` `McpSafeAssetSave(Asset);`.

The response then reports a save that did not happen:

- `:493` / `:600` / `:645` — `Result->SetBoolField(TEXT("saved"), Ctx.GetBool(TEXT("save"), true));` — the `saved` field merely **echoes the requested flag**, it is not a disk-presence probe.
- `:405` — `AddAssetVerification(Result, Asset);` sets `existsAfter:true` from the **asset registry**, not from disk.

The shared helper never writes anything —
`Source/PinWright/Private/Utils/AssetUtils.cpp:220-232`:

```cpp
bool McpSafeAssetSave(UObject* Asset)
{
    if (!Asset)
        return false;

    // UE 5.7+ Fix: Do not immediately save newly created assets to disk.
    // Saving immediately causes bulkdata corruption and crashes.
    // Instead, mark the package dirty and notify the asset registry.
    Asset->MarkPackageDirty();
    FAssetRegistryModule::AssetCreated(Asset);

    return true;
}
```

So the schema/database is registered (hence `existsAfter:true`, and `asset.dump`
reads it) but never lands on disk. The `save:true` default + `saved:true` +
`existsAfter:true` response together imply a persistence that did not happen, and
the response carries no `pendingFlush` / disk-presence signal to say otherwise.
`editor.save_all` is the only thing that actually flushes these dirty packages,
and nothing in the create responses tells the agent that is required.

## What it should do

A `save:true` create must persist to disk for real (or, if deferred, the response
must say so — never an unqualified `saved:true` / `existsAfter:true`). Mirror the
accepted sibling fixes (`B-metasound-create-save-no-disk-write` #2,
`B-audio-create-save-no-disk-write` #2, `B-niagara-save-no-disk-write` #2):

- Route the pose_search create/mutate `save:true` path through the in-tree
  real-save helper `SaveAssetToDiskReportingPresence` (wraps
  `SaveLoadedAssetThrottled` / `UEditorAssetLibrary::SaveLoadedAsset` +
  `IFileManager::FileSize` disk probe, gated by `ShouldTreatAssetSaveAsSuccess`)
  with `bForce=true`, instead of the mark-dirty-only `McpSafeAssetSave`. Fix at
  the single `FinishPoseSearchAsset` chokepoint (`:151-161`) plus the direct
  `HandleCreateSchema` save site (`:484-487`) so all three verbs are covered
  uniformly.
- Report an honest `saved` field gated on actual disk presence, plus a
  `pendingFlush:true` signal when the asset is dirty-only — instead of echoing
  the requested flag and an unqualified `existsAfter:true`.
- `UPoseSearchSchema` / `UPoseSearchDatabase` are **not** Blueprint/SCS assets, so
  the bulkdata-corruption vector that pins `McpSafeAssetSave` on Blueprint edits
  (`B-bp-saved-state-corruption-mcp-edits`) does not apply here — the shared
  corruption-sensitive `McpSafeAssetSave` and its other callers stay untouched.

## Workaround

After the create calls, run `editor.save_all` to flush the dirty packages to disk
before the editor closes or any `git reset --hard`. Nothing in the create
responses signals this is required.

severity rationale: impact=corruption × reach=rare -> Critical (cold-load-confirmed
silent asset loss on a reported-success save; per the rubric a write that loses
asset data is Critical regardless of the plugin-gated pose_search namespace's rare
reach).

## History
- `#2-go-fix` `IN-REVIEW` developer — GO, implemented and verified green. Rerouted all three `pose_search` `save:true` paths off the mark-dirty-only `McpSafeAssetSave` onto the real-save helper `SaveAssetToDiskReportingPresence(bForce=true)`, and replaced the bare `saved`-echo with the honest `AddAssetSaveReport(saveRequested/saved/pendingFlush)` contract: `FinishPoseSearchAsset` (the `create_database` + `add_database_animation` chokepoint) now returns the disk-presence bool, and `HandleCreateSchema`'s direct save site force-saves too. File: `Handlers/PoseSearch/PoseSearchHandler.cpp`. Full ticket scope (all three verbs), no split; the corruption-sensitive shared `McpSafeAssetSave` and its Blueprint/SCS callers stay untouched (`UPoseSearchSchema`/`UPoseSearchDatabase` are non-Blueprint). Severity Critical unchanged. Adopted the reporter's red test `PinWright.pose_search.CreateSchemaSaveWritesToDisk` (`Tests/Gameplay/TestPoseSearchCreateSaveWritesToDisk.cpp`) — observed failing pre-fix on the `.uasset` `IFileManager::FileSize` disk-presence probe, now `Result={Success}` post-fix (differential red→green; the schema genuinely lands on disk). Also fixed a pre-existing shared test-teardown bug surfaced by the run: `CleanupTestAsset` (`Tests/TestUtils.h`) force-deleted in-memory-only fixtures (`RF_Transient`, registered via `AssetCreated`, never on disk), erroring `Could not find the source asset` at Error level and failing BOTH `pose_search` tests; it now gates `DeleteAsset` on real on-disk presence. Both `PinWright.pose_search` tests green (2/2), plugin compiles clean.
- `#1-initial-repro` `OPEN` reporter — COLD-LOAD-confirmed asset loss from a REALISM-mode Motion Matching task. `pose_search.create_schema` (`/Game/Animation/MotionMatching/PS_Manny_Locomotion_Schema`, `UPoseSearchSchema` on `SK_Mannequin`, 3 Position channels root/foot_l/foot_r) and `pose_search.create_database` (`/Game/Animation/MotionMatching/DB_Manny_Locomotion`, bound to that schema, 4 Manny clips) both returned success with `saved:true` + `existsAfter:true`, and in-session `asset.dump` confirmed the schema binding/channels and the 4 sequence entries. A real editor cold restart (MCP quit discard=true, relaunch headless, gateway cold) then found BOTH assets ENTIRELY ABSENT on disk: `editor.open_asset` / `asset.validate` -> `[ASSET_NOT_FOUND]`, `asset.exists` -> exists:false, and the entire `Content/Animation/` tree missing on disk (no `*MotionMatching*` / `*Manny_Locomotion*` file). Editor stayed healthy (clean structured `ASSET_NOT_FOUND`), not a crash. Root cause: `PoseSearchHandler.cpp` routes every `save:true` through the shared mark-dirty-only `McpSafeAssetSave` — `HandleCreateSchema` (`:486`), `HandleCreateDatabase` (`:594` -> `FinishPoseSearchAsset` `:159`), `HandleAddDatabaseAnimation` (`:639` -> `:159`) — while `AddAssetVerification` (`:405`) sets `existsAfter:true` from the registry and the `saved` field (`:493`/`:600`/`:645`) merely echoes the requested flag. `McpSafeAssetSave` (`AssetUtils.cpp:220-232`) only `MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated()` then returns true — no package-save API. Same defect shape and same shared helper as `B-metasound-create-save-no-disk-write` / `B-niagara-save-no-disk-write` (both scope their fix to their own handlers and leave pose_search unfixed); `SaveAudioAsset` (audio) / `SaveMaterialAsset` (material) are the analogous per-namespace no-op helpers. The cold restart IS the replay-confirmation (asset loss the save-time integrity gate let through). Dedup: ripgrep over OPEN + closed found no pose_search / Motion Matching save-fidelity ticket — `F-pose-search-database-authoring` (DONE) added the namespace but does not cover the disk-write defect. Affected methods: `pose_search.create_schema`, `pose_search.create_database`, `pose_search.add_database_animation`. Fix: reroute the `FinishPoseSearchAsset` chokepoint + the `HandleCreateSchema` save site through `SaveAssetToDiskReportingPresence(bForce=true)` and report honest `saved` / `pendingFlush` (UPoseSearchSchema/Database are non-Blueprint, so the `McpSafeAssetSave` bulkdata-corruption deferral does not apply).
