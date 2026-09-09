---
id: B-model-compile-pie-blocked-save-unattributed
title: "model.compile with save:true rebuilds the UStaticMesh in memory before it meets the PIE save refusal, leaving the loaded asset ahead of its .uasset with no verb that reconciles them"
status: IN-REVIEW
severity: High
category: bug
tags: [model, compile, pie, savestate, diagnostics, error-payload, multi-agent, transient]
costly: 3
---

# `model.compile` builds first and refuses second, so a blocked write leaves memory ahead of disk

## The residual

The attribution half of this ticket is closed (see *What already landed*). What stayed open is
history entry `#2`'s ask: **check for an active PIE session BEFORE the mesh build and refuse the
whole compile up front.**

Before this fix the probe sat at the save chokepoint. The log order on every blocked compile was

```
LogStaticMesh: Built static mesh [0.05s] /Game/.../SM_WPN_Pistol.SM_WPN_Pistol
LogUtils: Error: The Editor is currently in a play mode.
```

— the mesh was rebuilt in place, its package left dirty, and only the write refused.

## Why the in-memory lead is the harm

The loaded `UStaticMesh` is then AHEAD of the bytes on disk, in a shared editor, with no verb that
reconciles them: `asset.reload` is the rollback verb and is a Critical editor-killer
(`B-asset-reload-access-violation-kills-editor`), and the asset cannot be persisted while PIE
holds. Two ways that goes wrong with nobody doing anything careless:

- another stream's `editor.save_all` or an `editor.quit {save:true}` persists a revision nobody in
  this session asked for and nobody reviewed;
- the editor dies and the rebuild is lost with the on-disk asset silently one revision behind its
  own committed `.pwmodel` source — the state that makes a provenance stamp claim something untrue.

Secondary cost: ~0.6 s of `BuildStaticMesh` plus a package create per attempt, all discarded.

The precedent is in the same verb at the same stage: the live-render-consumer guard refuses with
`MESH_REBUILD_CONSUMER_NOT_QUIESCABLE` before touching anything (`model.compile` § Live render
consumers: *"the compile is refused ... and nothing is built or saved"*), and a PIE probe is
cheaper than the consumer scan it already runs.

## Expected

`model.compile {save: true}` during PIE refuses before any geometry work, with the same error code
and the same report fields the save path already publishes — `PIE_ACTIVE`,
`saveState: "blockedByPie"`, `pendingFlush: false`, `pieActive`, `editorMode`, `pieWorlds` — and
nothing exists afterwards: no built mesh, no package, no `.uasset`. `save: false` must keep
working during a session; it asks for no write, so there is nothing for PIE to block.

## Repro

1. In a shared editor, start PIE (or have another agent start it).
2. `model.compile {filePath: "<any valid .pwmodel>", outputPath: "/Game/…", save: true}`.
3. Before: `success:true`, the mesh built, the package dirty, no `.uasset` on disk.
   After: `PIE_ACTIVE`, and `asset.exists` / the content browser show nothing was created.
4. Repeat with `save: false`: the compile still runs and the mesh exists in memory.

Observed 2026-09-02 / 09-03 on UE 5.8, on `SM_WPN_Pistol` (three times) and `SM_Prop_OilDrum`.

## What already landed (the primary ask, not this ticket's work)

Attribution — `saveState: "blockedByPie"` plus `pieActive` / `editorMode` / `pieWorlds` on the
`model.compile` response — was closed by the `asset.save` wave and reaches this verb through the
shared chokepoint: `Utils/AssetUtils.cpp` pre-checks PIE in `SaveLoadedAssetThrottled` and returns
`RefusedBlockedByPie`, maps it to `EAssetSaveState::BlockedByPie`, and `AddAssetSaveReport` emits
the block; `ModelCompileHandler.cpp` reaches that emitter via `PwModelCompiler` →
`GeometryAssetCreate`. Tickets: `B-asset-save-pie-failure-reports-pendingflush` and
`B-asset-save-omits-savestate-pie-block` (both IN-REVIEW), on the vocabulary
`B-editor-save-all-pie-diagnostic` (DONE) established.

## History
- `#4-bumped-by-cost` `OPEN` orchestrator — Severity Medium -> High by cost. Costly encounters counted: #1 (a completed mesh build lost its write; `saveState:"failed"` with no cause, so the caller could not tell a transient from a real error), #2 (same asset again, in-memory `UStaticMesh` left ahead of disk with no verb to reconcile, worked around by polling the editor log for the PIE-end marker and recompiling), #3 (hit twice in two minutes, and the same log carries the identical failure for three other agents' assets inside 25 minutes; establishing that the refused write would have been byte-identical needed a by-hand diff of twenty fields across two compile responses). Reach also applies: the encounters span the WEAPONS, VFX, ENV and PLAYER streams on one shared editor, and `model.compile` with a save is the ordinary way to make a `.pwmodel` edit stick.
- `#1-pie-cause-missing-from-model-compile` `OPEN` reporter — `model.compile` returned `success:true` with `saveState:"failed"` and no `.uasset` on disk while another agent's PIE session was running; `LogUtils: Error: The Editor is currently in a play mode.` was written to the log on the same call and dropped from the response. Distinct from `B-asset-save-omits-savestate-pie-block` and `B-asset-save-pie-failure-reports-pendingflush`, which are about `saveState` being absent — here it is present and only the *cause* is missing. Asks for `saveCause`/`pieActive`/`retryable` on the failure payload, or a pre-flight refusal, reusing the `ClassifyFailureReason` helper that `B-editor-save-all-pie-diagnostic` already landed for `editor.save_all`.
- `#2-inmemory-asset-left-ahead-of-disk` `OPEN` reporter — Hit again on the same asset, one grip-rake fix later, so the repro is stable across content: `model.compile` on `SM_WPN_Pistol` returned `success:true`, `assetTriangleCount:4812`, every health term clean, `saved:false`, `saveState:"failed"`, `pendingFlush:true`, and `ls` showed the `.uasset` unchanged at 211958 bytes with its old mtime. Root cause confirmed from the log rather than inferred: `LogPlayLevel: Creating play world package: /Game/FPS/Test/UEDPIE_0_T_Player` at `21:14:53`, my compile at `21:15:07`, and the VFX stream's `/Game/FPS/VFX/Emitters/E_Explosion_Debris` failing identically at `21:14:59` — one PIE session eating three streams' writes inside fifteen seconds. **What I want to add to this ticket is a consequence it does not currently name, and it is worse than the diagnostics gap.** The log line immediately before the failure is `LogStaticMesh: Built static mesh [0.05s] /Game/FPS/Weapons/Meshes/SM_WPN_Pistol.SM_WPN_Pistol` — so the compile DID rebuild the live `UStaticMesh` in memory and left its package dirty; only the write was refused. The loaded asset is therefore now AHEAD of the bytes on disk, in a shared editor, with no verb that reconciles them: the caller cannot roll the in-memory object back to the on-disk revision (`asset.reload` is the verb for that and it is a Critical editor-killer, `B-asset-reload-access-violation-kills-editor`), and cannot persist it while PIE holds. Two ways that goes wrong without anyone doing anything careless — another stream's save-all or an `editor.quit {save:true}` persists a revision nobody in this session asked for and nobody reviewed, or the editor dies (it has twice today) and the rebuild is lost with the on-disk asset silently one revision behind its own committed `.pwmodel` source, which is precisely the state that makes a `.pwmodel` provenance stamp claim something untrue. So the ask here is not only "name PIE in the payload". It is: **check for an active PIE session BEFORE the mesh build and refuse the whole compile up front**, the way the live-render-consumer guard already refuses with `MESH_REBUILD_CONSUMER_NOT_QUIESCABLE` before touching anything (`model.compile` § Live render consumers: "the compile is refused ... and nothing is built or saved"). That guard's existence is the precedent — same verb, same stage, same all-or-nothing reasoning — and a PIE check is cheaper than the render-consumer scan it already runs. Workaround I used, which is the one the format allows: leave the source on disk (it is the recoverable artefact), poll the log for the PIE-end marker, and recompile. Severity: I would argue this is understated at Medium now that the divergence is on the table, but leaving it as filed for the owner to judge.

- `#3-third-encounter-and-a-case-where-the-in-memory-lead-was-provably-harmless` `OPEN` reporter — Hit twice in two minutes at 2026-09-03 04:22 UTC on `Content/FPS/Weapons/Meshes/SM_WPN_Pistol.pwmodel`. `model.compile` returned `success: true`, a complete `health` / `bounds` / `parts` payload and `savedToDisk: false`, `saved: false`, `pendingFlush: true`, `saveState: "failed"`, `saveDetail: "The save was attempted and produced no durable revision; a flush will not help until the cause is cleared."` The actual cause is one line above it in the editor log and appears nowhere in the response:
```
[04.22.24:121] LogUtils: Error: The Editor is currently in a play mode.
[04.22.24:121] LogPinWrightSubsystem: Warning: SaveLoadedAssetThrottled: failed to save '/Game/FPS/Weapons/Meshes/SM_WPN_Pistol.SM_WPN_Pistol'
[04.22.24:121] LogPinWrightSubsystem: Warning: SaveAssetToDiskReportingPresence: ... state=failed outcome=Failed forced=true
    dirtyBefore=true existedBefore=true sizeBefore=87378 sizeAfter=87378
    stampBefore=2026-09-03T04:16:38.000Z stampAfter=2026-09-03T04:16:38.000Z
```
  Note `forced=true` and `dirtyBefore=true`: this is not the `only_if_is_dirty` no-op, it is a refusal, and the equal size/stamp pair is the proof that nothing was written.

  **It is not one caller's problem.** The same log carries the identical failure for three other agents' assets inside 25 minutes — `/Game/FPS/Env/Meshes/SM_ENV_Cabinet` and `SM_ENV_Desk` at 03:59, `/Game/FPS/Player/M_FPSArms` three times between 04:00 and 04:03. On a shared editor, ONE agent entering PIE silently costs every other stream its saves, and each of them gets a response that reads like a defect in their own asset. `saveDetail` saying "a flush will not help until the cause is cleared" without naming the cause is the sharpest part of the problem: it tells the caller to stop retrying and gives them nothing to act on.

  **One thing worth adding to `#2`'s in-memory-lead concern:** it is only dangerous when the compile CHANGED the geometry. Here it did not — the two refused compiles differed from the last durable one only in source comments, and `meshTriangleCount 2080`, `meshVertexCount 1276`, every `health` field, `bounds`, `materialSlotList` and all five `parts` entries were identical across all three. So the durable `.uasset` at `stamp 04:16:38` is the correct output of the current source and a per-triangle material-ID read-back taken from it is valid. A caller can only make that argument by hand, though, by diffing two compile responses. If the response carried a content hash of the built mesh, "the refused save would have written the same bytes" would be one comparison instead of twenty.

  Suggestion, in priority order: (1) name PIE in `saveDetail` / an `EDITOR_IN_PLAY_MODE` reason code — it is already known one call frame down; (2) say it is transient and that the remedy is another agent's `editor.stop`, not the caller's; (3) publish a built-mesh hash so a caller can tell a stale asset from an identical one.

- `#5-preflight-refusal-before-mesh-build` `IN-REVIEW` developer — Title and body narrowed to the residual; the ATTRIBUTION half was already closed by the `asset.save` wave (`B-asset-save-pie-failure-reports-pendingflush`, `B-asset-save-omits-savestate-pie-block`) on the vocabulary `B-editor-save-all-pie-diagnostic` (DONE) established, and verified reaching this verb through `Utils/AssetUtils.cpp` (`SaveLoadedAssetThrottled` PIE pre-check → `RefusedBlockedByPie` → `EAssetSaveState::BlockedByPie` → `AddAssetSaveReport`) via `Model/PwModelCompiler.cpp` and `Handlers/Geometry/GeometryAssetCreate.cpp`. Implemented `#2`'s residual: added `AddPieSaveRefusalReport` to `Source/PinWright/Private/Utils/AssetUtils.h`/`.cpp` — one probe through the existing `PinWrightPieSaveBlock` guard, publishing the SAME payload the post-hoc path emits (`saveRequested`, `saved:false`, `pendingFlush:false`, `saveState:"blockedByPie"`, `saveDetail`, `pieActive`, `editorMode`, `pieWorlds`) so one caller vocabulary covers both timings — and called it from `Source/PinWrightGeometry/Private/Handlers/Model/ModelCompileHandler.cpp` when `save:true`, ahead of `RunGuardedStaticMeshRebuild`, so the safe-point hop, the consumer quiesce and the mesh build never start; refusal is `PIE_ACTIVE` with `assetPath` / `sourcePath`, the same code `asset.save` uses. `save:false` is deliberately NOT refused: it requests no write. The `save` parameter description and `docs/wiki-src/model.md` / `docs/wiki-src/safe-mutation-save.md` state the new contract. Regression tests in `Source/PinWrightGeometry/Private/Tests/Model/TestModelCompilePieRefusal.cpp`: `PinWright.Model.Handlers.CompileRefusesBeforeBuildingWhilePieHoldsTheEditor` and `PinWright.Model.Handlers.CompileWithoutSaveIsNotRefusedDuringPie`, both scoping `GIsPlayInEditorWorld` (one half of the engine's own two-global predicate) rather than starting a live session. Counterfactual: revert the pre-flight in `ModelCompileHandler.cpp` and the first test fails on three assertions — the response is `success:true` instead of `PIE_ACTIVE`, and `FindObject<UStaticMesh>` / `FindPackage` return the mesh and package the compile built before the save was refused, which is exactly the in-memory lead this entry is about. Sibling verbs checked: `blueprint.compile_bpir` already refuses up front (`BpirCompilerHandler.cpp`, unconditional `IsPlayInEditorActive` gate) so it does not share the residual; `material.compile_mgir` DOES share it (an `Append` compile tears down and rebuilds the target graph before the refused write) but is left alone — `B-compile-mgir-pie-blocked-save-hard-errors` is IN-REVIEW and its landed fix deliberately keeps the compile result rather than refusing, so changing it here would contradict a decision under review. `MeshOpsHandler.cpp`'s create-and-save verbs are a third, unticketed surface with the same shape.
