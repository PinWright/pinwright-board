---
id: B-set-view-mode-exec-failed
title: "editor.set_view_mode returns EXEC_FAILED; viewmode exec never reaches the active level viewport"
status: DONE
severity: Medium
category: bug
tags: [editor, viewport, set_view_mode, viewmode, exec, wireframe]
---

# editor.set_view_mode returns EXEC_FAILED; viewmode exec never reaches the active level viewport

`editor.set_view_mode {viewMode:"Wireframe"}` returns `[EXEC_FAILED] View mode command failed`,
despite the handler docs claiming Wireframe is recognised. Alias resolution itself is fine
(`Wireframe` maps correctly); the failure is in dispatch.

Source (`ViewportHandler.cpp` L138-169):
- The handler builds `viewmode Wireframe` and calls `GEditor->Exec(nullptr, *Cmd)` (L159-160).
  When `GEditor->Exec` returns `false` the handler emits `EXEC_FAILED` (L167).
- `viewmode` is a per-viewport-client console command — it is consumed by the active
  `FLevelEditorViewportClient`, not by a world or by `UEditorEngine::Exec`'s global handlers.
  Passing a `nullptr` world (and never routing the exec at the active level viewport client)
  means `GEditor->Exec` finds no handler that consumes it and returns `false`. Unlike
  `set_camera`/`set_game_view` (which use subsystem/world targets), this handler never resolves
  the active level viewport, so the command both fails and would target the wrong client even
  if it "succeeded".

This is distinct from `B-set-camera-no-viewport-redraw` (where set_camera returns `success:true`
but the captured frame is stale): there the exec succeeds and the bug is a missing force-redraw;
here the exec returns an outright failure. The shared theme is "viewport commands not reaching
the active level viewport client" — cross-referenced, filed separately because the symptom and
fix differ.

**Workaround:** `editor.console_command {command:"viewmode wireframe"}` reports `success:true`,
but the change may not appear in a subsequent capture (same wrong-viewport-target / stale-frame
issue as `B-set-camera-no-viewport-redraw`); not reliable for screenshot verification.
**Fix:** resolve the active level viewport client (e.g. `FLevelEditorModule::GetFirstActiveViewport()`
→ `GetAssetViewportClient()`, matching the pattern in `set_viewport_realtime` L184-199) and apply
the view mode directly via `SetViewMode(EViewModeIndex)`, then `Invalidate()`/force a redraw. This
both fixes EXEC_FAILED and guarantees the correct viewport is targeted before any capture.

## History
- `#1-initial-repro` `OPEN` reporter — `editor.set_view_mode {viewMode:"Wireframe"}` → `{"error":"[EXEC_FAILED] View mode command failed"}`. Fallback `editor.console_command {command:"viewmode wireframe"}` → `{success:true}` but the next HighResShot still showed a Lit frame. Source (ViewportHandler.cpp L159-167): handler does `GEditor->Exec(nullptr, "viewmode Wireframe")`, which returns false because `viewmode` is a per-viewport-client command that this call never routes to the active level viewport; on false it emits EXEC_FAILED. Distinct symptom from B-set-camera-no-viewport-redraw (which returns success but stale); cross-referenced.
- `#2-resolve-viewport-client` `IN-REVIEW` developer — Rewrote the `editor.set_view_mode` handler in `ViewportHandler.cpp` to resolve the active level viewport via `FLevelEditorModule::GetFirstActiveViewport()` → `GetAssetViewportClient()` and apply the mode with `SetViewMode(EViewModeIndex)` + `Invalidate()` (matching `set_viewport_realtime`), mapping each alias to its `VMI_*` index; kept the console-command exec as a fallback but now routes it through the editor world instead of `nullptr`, and rejects unknown modes with `UNKNOWN_VIEW_MODE`. Added regression test `Tests/EditorOps/SetViewModeExecTest.cpp` (`FEditorSetViewModeExecTest`) asserting `bSuccess==true` and `viewMode=="Wireframe"` when an active viewport is present. Did not compile.
- `#3-review-disposition` `IN-REVIEW` developer — Reviewer findings triaged, no code change warranted. (a) `Wireframe`→`VMI_BrushWireframe` mapping confirmed correct against UE 5.6 `EngineBaseTypes.h:959` (`VMI_BrushWireframe UMETA(DisplayName="Wireframe")`; `VMI_Wireframe` is "CSG Wireframe"); `EViewModeIndex` reaches the handler via the existing `EngineBaseTypes.h` include. (b) Regression test confirmed to invoke the production handler through `InvokeHandlerWithCapture` (real `Tests/TestUtils.h` helper) — kept as-is. (c) Reviewer "drive-by changes" finding (set_camera `ForceRedrawActiveViewport`, screenshot `CaptureActiveLevelViewportToScreenshot`) is a shared-working-tree artifact: those edits are the live `IN-REVIEW` implementations of `B-set-camera-no-viewport-redraw #2` and `F-editor-viewport-screenshot #2` and were left intact (reverting them would destroy other tickets' work). (d) "Untracked test file" / "unstaged board file" findings are commit-staging actions outside this fix pass's permitted scope (`git add`/`git commit` not run); the artifacts themselves are present and correct, to be staged by the commit step alongside the other two tickets.
- `#4-rereview-scope-conflict` `IN-REVIEW` developer — Re-review escalated the three findings from `#3`. Verified each against source and the sibling board files, then held position on the two unactionable ones with a surfaced conflict; resolved the actionable one. (1) The flagged "drive-by" code is confirmed owned by other live tickets: `ForceRedrawActiveViewport()` (`ViewportHandler.cpp` L60-72) + its `set_camera` calls = `B-set-camera-no-viewport-redraw #2/#3`; `CaptureActiveLevelViewportToScreenshot()` (L79-127) + its `editor.screenshot` use = `F-editor-viewport-screenshot #2/#3`; `EditorHandlerUtils::EditorToggleParamOpt/ToggleKeys` in `set_viewport_realtime`/`set_game_view` (L283/291/326/329) = the editor-toggle-param refactor. (2) The three includes the reviewer wants removed are each used *only* by that sibling code: `PreviewViewportCaptureUtils.h`→`CaptureEditorViewportToPng` (L123); `SceneViewport.h`→`FSceneViewport`/`GetSharedActiveViewport` (L102-103); `EditorHandlerUtils.h`→both toggle handlers. Removing them WITHOUT also reverting those handlers' bodies would not compile, and reverting the bodies would delete two IN-REVIEW tickets' shipped work in this single uncommitted file — out of scope for this ticket and a worse outcome than the per-line scope overlap. So findings (1)/(2) are held as a shared-working-tree artifact, not actioned. Only `EngineBaseTypes.h` (L24, this ticket's genuinely-required include) and the `set_view_mode` handler (L218-278) belong to this ticket; both are correct and unchanged. (3) Untracked-test finding: `git add`/`git commit` are explicitly forbidden in this pass, so per the reviewer's stated alternative, staging of `Tests/EditorOps/SetViewModeExecTest.cpp` (real production-invoking regression test, kept as-is) is explicitly deferred to a separate commit step alongside the sibling tickets. No production-code edits made this pass.
- `#5-verify-fix` `DONE` tester — Verified live: `editor.set_view_mode {viewMode:"Wireframe"}` → `{"success":true,"viewMode":"Wireframe"}` (previously `[EXEC_FAILED]`). `{viewMode:"BogusMode"}` → `[UNKNOWN_VIEW_MODE] Unrecognised view mode 'BogusMode'` (confirms #2's reject path). `{viewMode:"Lit"}` → `{"success":true,"viewMode":"Lit"}` (viewport restored). EXEC_FAILED no longer reproduces.
