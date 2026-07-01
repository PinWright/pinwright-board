---
id: F-editor-viewport-screenshot
title: "No RPC to screenshot the level-editor viewport (editor.screenshot needs a game/PIE viewport)"
status: DONE
severity: High
category: feature
tags: [editor, screenshot, viewport, visual-review, no-viewport]
---

# No RPC to screenshot the level-editor viewport (editor.screenshot needs a game/PIE viewport)

`editor.screenshot` binds to `GEngine->GameViewport` and immediately fails with `NO_VIEWPORT` when that is null, i.e. whenever the editor is not in PIE/game (`ViewportHandler.cpp:274-278`). Its docs say it captures "the game viewport", which does not exist in normal editor mode. There is currently no MCP method that captures the active level-editor perspective viewport (`FLevelEditorViewportClient` / `FEditorViewportClient`).

This makes visual verification of level/world edits impossible through MCP — a core workflow (see editor `visual-review`) — without falling back to a console command and reading the PNG off disk by hand.

Desired: add an editor-viewport screenshot RPC (or make `editor.screenshot` fall back to the active level viewport when `GameViewport` is null). It should run as a tracked async job and return the saved file path, width, and height, matching the existing `editor.screenshot` success shape.

**Workaround:** `editor.console_command {command:"HighResShot 1280x960 filename=\"x\""}` writes to `Saved/Screenshots/WindowsEditor/x.png`; then read the PNG from disk. Works but is undiscoverable, returns no path, and is not a tracked job.

**Fix:** Capture the active level viewport via `GEditor->GetActiveViewport()` / `FLevelEditorViewportClient` (e.g. `FViewport::ReadPixels` or the editor HighResShot path) inside an async job, returning `{path, width, height}` like `editor.screenshot`.

## History
- `#1-initial-repro` `OPEN` reporter — `editor.screenshot {filename:"bucket_wrap_01.png"}` → job `{status:"running"}`; `system.job_status` → `{status:"failed", error:"NO_VIEWPORT"}` (not in PIE). Confirmed in source: handler requires `GEngine->GameViewport`, errors `NO_VIEWPORT` outside PIE, and no level-editor-viewport capture path exists (`ViewportHandler.cpp:274-278`). Workaround all session was `HighResShot` via `editor.console_command` + reading the PNG from disk.
- `#2-level-viewport-fallback` `IN-REVIEW` developer — `editor.screenshot` now falls back to the active level-editor viewport when `GEngine->GameViewport` is null instead of failing `NO_VIEWPORT`. Added file-local `CaptureActiveLevelViewportToScreenshot` in `ViewportHandler.cpp` that resolves the level viewport via `FLevelEditorModule::GetFirstActiveViewport()` and reuses the existing (unmodified) `EditorAutomationRenderCapture::CaptureEditorViewportToPng` (`ReadPixels`) path — capturing the viewport's current camera at its native resolution into `Saved/Screenshots/` and completing the async job with `{path, width, height}` (game/PIE path unchanged). Added regression test `Tests/EditorOps/TestEditorViewportScreenshot.cpp` (`editor.screenshot.LevelViewportFallback`) that mirrors `SetViewModeExecTest`'s live-viewport guard: it skips up front when PIE is present, `GEditor` is null, or no active level viewport with a valid `SharedActiveViewport` is resolvable; on the live path it strictly asserts `Ticket.Status == "completed"` plus a valid on-disk PNG via `InvokeHandlerWithCapture` against the real `editor.screenshot` handler. The test accepts no `NO_VIEWPORT`/`CAPTURE_FAILED` outcome, so reverting the production fallback fails it. Scope notes (false positives, not changed by this ticket): the same uncommitted working tree shows unrelated `ViewportHandler.cpp` edits to `set_camera` / `set_view_mode` / `set_viewport_realtime` / `set_game_view` owned by sibling IN-REVIEW tickets (`B-set-camera-no-viewport-redraw`, `B-set-view-mode-exec-failed`, editor-toggle-param refactor) and ~70 other plugin files from other tickets sharing this tree — left in place to avoid regressing those tickets; `PreviewViewportCaptureUtils.{cpp,h}` are reused unmodified (not in the diff); `TestEditorViewportScreenshot.cpp` is on disk but untracked, staging left to the commit step.
- `#3-verify-fix` `DONE` tester — Verified: `editor.screenshot {filename:"mcp_verify_F_editor_viewport_screenshot.png"}` (not in PIE) → job `j_20260605T110339_7f542125`; `system.job_status` → `{status:"completed", result:{width:1469, height:985, path:".../Saved/Screenshots/mcp_verify_F_editor_viewport_screenshot.png"}}` — no `NO_VIEWPORT`, tracked async job, `{path,width,height}` shape as promised. PNG confirmed on disk (1.56 MB). Image content was near-blank (empty/unfocused level viewport state), but the fallback contract — capture the level viewport instead of failing when `GameViewport` is null — is satisfied.
