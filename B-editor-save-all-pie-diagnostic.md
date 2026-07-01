---
id: B-editor-save-all-pie-diagnostic
title: "editor.save_all error payload omits PIE-lock cause and per-asset failure reasons"
status: DONE
severity: Medium
category: bug
tags: [editor, save-all, pie, diagnostics, error-payload, ergonomics]
---

# editor.save_all error payload omits PIE-lock cause and per-asset failure reasons

When PIE is running, dirty assets that PIE has locked (e.g. a Widget Blueprint
edited mid-session via `blueprint.compile_bpir`) cannot be saved. `editor.save_all`
correctly reports `success:false`, but the payload gives no actionable diagnostic:

```json
{"success": false, "savedCount": 0, "totalDirty": 1, "error": "Saved 0 of 1 dirty assets."}
```

After PIE was manually stopped, the same call against the same dirty asset
returned `{success:true, savedCount:1, totalDirty:1}`. The asset (Widget BP
`W_HUD_RaceTrackEnd`) was legitimately blocked by UE's PIE asset lock — not a
generic save failure, not source control, not read-only, not a compile error.

The caller had to guess between PIE / source-control / read-only / checkout /
compile-error and try several remedies before landing on "stop PIE".

## Proposed payload additions

On failure, include the environmental cause and a per-asset breakdown:

```json
{
  "success": false,
  "savedCount": 0,
  "totalDirty": 1,
  "pieActive": true,
  "editorMode": "PIE",
  "failedAssets": [
    {"path": "/Game/UI/W_HUD_RaceTrackEnd", "reason": "BlockedByPie"}
  ],
  "error": "Saved 0 of 1 dirty assets (PIE active; 1 asset locked by PIE)."
}
```

Reason enum candidates: `BlockedByPie`, `ReadOnly`, `SourceControlCheckedOut`,
`CompileError`, `Unknown`. `pieActive` is cheap to read via
`GEditor->PlayWorld != nullptr` / `GEditor->IsPlayingSessionInEditor()`.

**Workaround:** Caller checks PIE state independently and stops PIE before retry.

**Fix:** In `SaveAllHandler.cpp`, before reporting completion, probe PIE state
and inspect the not-saved package set to attribute a reason per asset. Pass the
enriched payload through `CompleteJob`.

## History
- `#1-pie-locked-saves-opaque` `OPEN` reporter — Two `editor.save_all` calls during PIE returned `{success:false, savedCount:0, totalDirty:1, error:"Saved 0 of 1 dirty assets."}`. After user stopped PIE, the same call succeeded. Payload should surface `pieActive`/`editorMode` and a `failedAssets` list with per-asset reasons (e.g. `BlockedByPie`) so callers don't have to guess between PIE / source-control / read-only / compile-error.
- `#2-enriched-payload-and-helper` `IN-REVIEW` developer — Added `EditorSaveAllDiagnostic::BuildSaveAllResultJson` + `ClassifyFailureReason` (new `Private/Handlers/Editor/EditorSaveAllDiagnostic.h`); rewired `editor.save_all` in `EditorCommandHandler.cpp` to probe `GEditor->PlayWorld` and emit `pieActive`/`editorMode`/`failedAssets[]` plus a PIE-aware error message. Regression: `EditorAutomationRpcGateway.editor.save_all.ResultJsonExposesPieActive` (TestEditorHandlers.cpp).
- `#3-verify-pie-diagnostics` `DONE` tester — Verified: `system.run_tests` exact test `EditorAutomationRpcGateway.editor.save_all.ResultJsonExposesPieActive` resolved and completed with `has_errors:false`, `missingTests:[]`, proving the enriched `editor.save_all` result JSON regression passes.
