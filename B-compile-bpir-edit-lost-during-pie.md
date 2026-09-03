---
id: B-compile-bpir-edit-lost-during-pie
title: "A compile_bpir edit made while PIE is running can be lost with no trace: compiled:true is returned, the package is later NOT dirty, and save_all reports nothing to save"
status: IN-REVIEW
severity: High
category: bug
tags: [blueprint, compile_bpir, pie, save, silent-data-loss, save_all, multi-agent]
encounters: 1
lastSeen: 2026-09-02T20:08:00Z
---

# An edit made during PIE disappears, and every readback says the package is clean

## What happened

Three Behavior Tree task Blueprints got the same one-event addition (`ReceiveAbortAI`) through three
consecutive `blueprint.compile_bpir` calls, in one batch, while **another agent's PIE session was
running** in the shared editor. All three answered identically:

```
{"nodeCount":5,"createdNodes":[...],"errors":[],"warnings":[],"compiled":true,"status":"UpToDate","success":true}
```

Minutes later, with PIE stopped, `editor.save_all` answered
`{"success":true,"savedCount":14,"totalDirty":14}` — a clean, complete save with nothing skipped and
nothing reported as failed.

Only **one** of the three edits reached disk. Checked against the files, not against the editor:

```
$ grep -ac ReceiveAbortAI BTT_PeekFire.uasset   -> 0     (mtime unchanged, 22:41)
$ grep -ac ReceiveAbortAI BTT_Suppress.uasset   -> 2     (mtime 23:07, saved by save_all)
$ grep -ac ReceiveAbortAI BTT_Reload.uasset     -> 0     (mtime unchanged, 22:41)
```

The two lost ones were not in `save_all`'s dirty set at all — `totalDirty: 14` did not include them.
So between the successful compile and the save, their packages went from "edited" to "clean" without
being written. Nothing in any response says so.

Re-running the identical `compile_bpir` after PIE ended, then `asset.save {force:true}`, put the
handler on disk immediately (`grep -ac` -> 2 for all three), so the BPIR itself was never the
problem.

## Why this is severe

Every signal available to the caller says the work landed:

- `compile_bpir`: `compiled:true`, `status:"UpToDate"`, `errors:[]`, `warnings:[]`, a list of
  created node GUIDs;
- `editor.save_all`: `success:true`, `savedCount == totalDirty`, no `failedAssets`;
- a read-back through `blueprint.get` / `blueprint.decompile_function` would also have shown the
  event, because those read the in-memory object.

Only a byte-level check of the `.uasset` — which is the project's own `CLAUDE.md` rule, and the
only reason this was caught — disagrees. In a shared editor where another stream can start PIE at
any moment, this makes any batch of edits silently lossy, and the loss is *partial* (one of three
survived), so a spot-check of one asset would have confirmed the wrong conclusion.

The 22:41 mtimes are the same ones `editor.save_all` had produced in an earlier flush, which
suggests the packages were reloaded from disk (dropping the edit and clearing the dirty flag)
rather than never marked dirty — PIE start/stop reinstances Blueprints, and a BT task Blueprint is
instanced into the running tree.

## Expected

- If PIE is going to discard in-memory Blueprint edits, `compile_bpir` should refuse while a PIE
  session is active (`PIE_ACTIVE`, the way `editor.save_all` already refuses with
  `"reason":"BlockedByPie"`), or persist the edit before PIE can reclaim it.
- Failing that, `compile_bpir` should report `pieActive: true` in its response so a caller can
  re-verify, and `save_all`'s `totalDirty` must not silently shrink between a successful edit and
  the save.

`editor.save_all` already knows about this hazard — during the same session it returned
`[SAVE_FAILED] Saved 0 of 24 dirty assets (PIE active; 24 asset(s) locked by PIE)` with a per-asset
`BlockedByPie` reason. `compile_bpir` has no equivalent guard.

**Workaround:** never author while another stream's PIE is running; after every `compile_bpir`,
`grep -a` the `.uasset` for a token from the new graph and re-issue the compile plus
`asset.save {force:true}` if it is absent.

severity rationale: impact=silent data loss on a normal path, with every readback agreeing that the
work succeeded x reach=every multi-agent session (any other stream's PIE run overlaps ordinary
authoring) -> High

## Fix

The handler had no general play-mode preflight: its live-generated-class survey only covered instances that might be reinstanced after the Blueprint was loaded, so `GIsPlayInEditorWorld` could still reach BPIR compilation and mutation. `compile_bpir` and both `insert_bpir_*` verbs now check `PinWrightPieState::IsPlayInEditorActive()` after path/code validation and before Blueprint loading, pre-compilation, transactions, or graph changes, returning `ErrorCodes::ERR_PIE_ACTIVE`; the `allowReinstancing` opt-in cannot bypass this refusal.

Files changed:

- `Plugins/PinWright/Source/PinWright/Private/Handlers/Blueprint/BpirCompilerHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Handlers/ErrorCodes.h`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Blueprint/TestBpirCompilePieGuard.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Core/TestErrorCodeRegistry.cpp`
- `Plugins/PinWright/Docs/wiki-src/blueprint.md`

Regression test: `PinWright.blueprint.compile_bpir.RefusesDuringPie` invokes the registered production handler with a valid BPIR payload while `GIsPlayInEditorWorld` is guarded true, asserting `PIE_ACTIVE`, response delivery, and unchanged graph/package state.

Deliberate non-changes: `save_all` was not modified because its `totalDirty` value is a snapshot at invocation and is not separately proven to cause this defect; `Utils/PieState.h/.cpp` and `EditorCommandHandler` were not changed. The `ErrorCodes` ratchet was completed for the whole handler by registering only `BLUEPRINT_COMPILE_FAILED` and `INTEGRITY_FAILURE` and deleting their stale quarantine entries; no partial-adoption exception was added. Successful non-PIE BPIR authoring remains in-memory-only and is persisted through the existing `asset.save` path after PIE stops.

## History
- `#1-filed` `OPEN` reporter — Found while adding BT task abort handlers for the FPS AI stream in a
  shared UE 5.8 editor (`EAContentExamples58`) during the WEAPONS stream's PIE run. Evidence above is
  verbatim: three identical `compile_bpir` successes, one `editor.save_all` reporting
  `savedCount:14, totalDirty:14`, and `grep -ac ReceiveAbortAI` over the three `.uasset` files giving
  `0 / 2 / 0`. The surviving one (`BTT_Suppress`) is the one whose package was still dirty at save
  time; the two lost ones kept their earlier 22:41 mtimes. Recovery was a plain re-compile plus
  `asset.save {force:true}` once PIE had stopped, verified again by `grep -ac` -> `2 / 2 / 2`. No
  plugin source read. Related in spirit to the project's "verify a write against disk" rule, but
  this is a tool-side gap: the plugin has the PIE information (`editor.save_all` uses it) and does
  not surface it on the authoring path.
- `#2-pie-active-guard` `IN-REVIEW` developer — Added the pre-load `PIE_ACTIVE` refusal, structural regression coverage, and the compile_bpir contract update; no save_all change.
