---
id: B-model-compile-pie-blocked-save-unattributed
title: "model.compile builds the mesh, writes no .uasset and reports saveState:failed without naming PIE as the cause, so the caller cannot tell a 30-second transient from a real error"
status: OPEN
severity: High
category: bug
tags: [model, compile, pie, savestate, diagnostics, error-payload, multi-agent, transient]
costly: 3
---

# `model.compile` does not attribute a PIE-blocked save, so a transient reads as a hard failure

## Symptom

In a shared editor where another agent was cycling PIE sessions, `model.compile` on a source
that had already validated clean returned, twice, identically:

```json
{"success": true,
 "assetPath": "/Game/FPS/Weapons/Meshes/SM_WPN_Pistol",
 "assetTriangleCount": 4932, "assetVertexCount": 6385,
 "health": {"isClosed": true, "boundaryEdges": 0, "selfIntersections": 0, ...},
 "savedToDisk": false,
 "saveRequested": true, "saved": false, "pendingFlush": true,
 "saveState": "failed",
 "saveDetail": "The save was attempted and produced no durable revision; a flush will not
                help until the cause is cleared. The editor log carries a
                SaveAssetToDiskReportingPresence line with the outcome, sizes and timestamps."}
```

No `.uasset` existed on disk afterwards — confirmed by `ls`, which is the check
`CLAUDE.md` § "Verify a write against disk" exists for and which is what caught this.

## What is right here, and what is missing

**`saveState` IS returned.** This is NOT the `asset.save` field-loss bug — see
`B-asset-save-omits-savestate-pie-block` and `B-asset-save-pie-failure-reports-pendingflush`,
which are about the field being dropped entirely. `model.compile` returns `saveState: "failed"`
and a `saveDetail` that correctly says a flush will not help. That much works.

**What is missing is the cause.** The blocker was another session's PIE, and the editor knew it
on that very call. From `Saved/Logs/EAContentExamples58.log`, the three lines are consecutive:

```
LogStaticMesh: Built static mesh [0.04s] /Game/FPS/Weapons/Meshes/SM_WPN_Pistol.SM_WPN_Pistol
LogUtils: Error: The Editor is currently in a play mode.
LogPinWrightSubsystem: Warning: SaveLoadedAssetThrottled: failed to save '/Game/.../SM_WPN_Pistol'
LogPinWrightSubsystem: Warning: SaveAssetToDiskReportingPresence: '...' is NOT durable after this
  save. state=failed outcome=Failed forced=true dirtyBefore=true existedBefore=false ...
```

`LogUtils: Error: The Editor is currently in a play mode.` never reaches the response. The
caller is told "the cause" must be cleared and is sent to the log to find out what it is.

## Why the distinction matters more here than elsewhere

A PIE-blocked `model.compile` is **transient and self-clearing**, and it is the only common
cause that is. Measured on this editor: PIE sessions ran ~70 s with ~30 s gaps between them, and
the identical compile succeeded with `saveState: "written"` on the first attempt inside a gap —
no argument changed. So the correct response to `BlockedByPie` is "wait and retry", and the
correct response to every other `failed` cause (read-only file, source control, a package the
asset registry will not create) is "stop and fix something". `saveDetail` gives the same
sentence for both.

Without attribution the caller either gives up on a compile that would have worked 30 seconds
later, or writes a blind retry loop against causes that never clear. Both were live options
here; the log dig is what settled it, and a log dig is not something a caller should need to do
to decide whether to retry.

Second cost: the mesh is fully built before the save is attempted — 0.6 s of `BuildStaticMesh`
plus a package create, per attempt, all of it discarded. A pre-flight PIE probe would refuse in
microseconds and say why.

## The fix pattern already exists in this plugin

`B-editor-save-all-pie-diagnostic` is `DONE` and added exactly this to `editor.save_all`:
`EditorSaveAllDiagnostic::BuildSaveAllResultJson` + `ClassifyFailureReason`, probing
`GEditor->PlayWorld` and emitting `pieActive` / `editorMode` / a per-asset `reason`. Its own
notes say `pieActive` is cheap: `GEditor->PlayWorld != nullptr` /
`GEditor->IsPlayingSessionInEditor()`. The same classifier applied on the `model.compile` save
path would close this.

## Expected

Either of these; the first is cheaper for the caller, the second cheaper for the editor.

1. Carry the cause into the existing failure payload:

```json
{"saveState": "failed", "saveCause": "BlockedByPie", "pieActive": true, "editorMode": "PIE",
 "retryable": true,
 "saveDetail": "The editor is in play mode, so no package can be written. This clears when PIE
                stops; the compiled mesh was discarded. Retry the compile then."}
```

2. Pre-flight: refuse before building, with the same fields, so a 0.6 s mesh build is not spent
   on a write that cannot land.

`retryable` (or the `saveCause` enum a caller can switch on) is the field that does the work.
`BlockedByPie` is the only value of it that means "wait"; every other `failed` cause means
"stop".

## Repro

1. In a shared editor, start PIE (or have another agent start it).
2. `model.compile {filePath: "<any valid .pwmodel>", outputPath: "/Game/…"}`.
3. Response: `success:true`, `saveState:"failed"`, no mention of PIE. `ls` the target path — no
   `.uasset`.
4. Stop PIE. Re-run the identical call. `saveState:"written"`, file on disk.

Observed 2026-09-02 on UE 5.8, on `SM_WPN_Pistol` (twice) and `SM_Prop_OilDrum` (once), all
three cleared on the first retry taken inside a PIE gap.

## Workaround

Poll `Saved/Logs/EAContentExamples58.log` for the most recent
`PIE: Server logged in` versus `LogWorld: BeginTearingDown for …UEDPIE…` and compile only when
the latter is more recent. That is what was done here; it worked on the first attempt for all
four remaining assets, but it requires log access and knowledge of two log strings that are not
documented anywhere as the PIE-state probe.

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
