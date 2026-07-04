---
id: B-set-game-view-keeps-editor-billboards
title: "editor.set_game_view reports success but never enables game view; editor billboards persist in captures"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, viewport, set_game_view, game-view, exec, false-success, billboard, screenshot]
---

# editor.set_game_view reports success but never enables game view; editor billboards persist in captures

`editor.set_game_view {enabled:true}` returns `{"success":true,"gameViewEnabled":true}`, but a subsequent
`editor.screenshot_window` of the main editor still shows the DirectionalLight / SkyLight / SkyAtmosphere
editor billboard sprites that real Game View (the G key) hides. Silent false-success: the readback claims
game view is on, so the caller trusts a lie and can never get an icon-free viewport shot.

Root cause (`ViewportHandler.cpp` L332-333): the handler runs
`GEditor->Exec(World, "ToggleGameView 1"/"ToggleGameView 0")` and never checks the return. `ToggleGameView`
is **not** an engine exec/console command — it exists only as a G-key `FUICommandInfo`
(`LevelViewportActions.cpp:43`) mapped to `SLevelViewport::ToggleGameView` (`SLevelViewport.cpp:2791-2798`),
which calls `FEditorViewportClient::SetGameView(bool)`. No `Exec` handler parses `TOGGLEGAMEVIEW` anywhere in
UE 5.6, so the call is a no-op returning `false`; game view is never toggled on any viewport, yet the handler
unconditionally reports `gameViewEnabled:true`. `editor.screenshot_window` (L762) composites the real window
via `FSlateApplication::TakeScreenshot`, so it WOULD render icon-free if game view were actually on — it isn't.

Same "viewport command never reaches the captured level viewport" family as `B-set-view-mode-exec-failed`
and `B-set-camera-no-viewport-redraw`; distinct symptom + fix, filed separately and cross-referenced.

**Workaround:** move the icon actors (RTSun/RTSkyLight/RTSky) out of the camera frame via `actor.set_transform` before capturing.
**Fix:** resolve the active level viewport (`FLevelEditorModule::GetFirstActiveViewport()` →
`GetAssetViewportClient()`, the pattern `set_view_mode`/`set_viewport_realtime` already use) and call
`SetGameView(bEnabled)` directly (or `ULevelEditorSubsystem::EditorSetGameView`) + `Invalidate()`; report the
real `IsInGameView()` state instead of an unconditional `gameViewEnabled:true`.

severity rationale: impact=silent false-success (returns success:true+gameViewEnabled:true while game view stays off; caller builds captures on a false readback) × reach=capture-prep verb, not every-session but not a rare edge → High, matching the sibling silent-false-success `B-set-camera-no-viewport-redraw` (High) over the honest-error `B-set-view-mode-exec-failed` (Medium).

## History
- `#1-initial-repro` `OPEN` reporter — `editor.set_game_view {enabled:true}` → `{"success":true,"gameViewEnabled":true}` (repeated across the session, always true), yet `editor.screenshot_window {window_title:"PDS - Unreal Editor"}` after a `set_view_mode Lit` + a spawned scene still showed the SkyLight/SkyAtmosphere dome sprites over the geometry; moving RTSun/RTSkyLight/RTSky to z=5000 via `actor.set_transform` removed the icons, confirming they were actor billboards game view failed to hide. Root cause verified against UE 5.6 source: handler does `GEditor->Exec(World, "ToggleGameView 1")` (`ViewportHandler.cpp:332-333`) but `ToggleGameView` is not an exec command (only a G-key UI command → `SLevelViewport::ToggleGameView` → `FEditorViewportClient::SetGameView`); the exec returns false, the handler ignores it and reports success. Fix: resolve the active level viewport and call `SetGameView` directly, then report the real `IsInGameView()`.
- `#2-fix-set-game-view` `IN-REVIEW` developer — Confirmed defect in the host's UE 5.7 source (`Handlers/Editor/ViewportHandler.cpp:332-337`): phantom `GEditor->Exec("ToggleGameView 1"/"0")` (no exec handler parses `TOGGLEGAMEVIEW` — grep across `C:\UE_5.7\Engine\Source` shows it only as the `LevelViewportActions.cpp:43` G-key `FUICommandInfo`, never an `FParse::Command`) ignored + unconditional `gameViewEnabled:bEnabled`. Fix: resolve the active level viewport client via `FLevelEditorModule::GetFirstActiveViewport()->GetAssetViewportClient()` (the same pattern the sibling `set_view_mode`/`set_viewport_realtime` handlers use), with a `GEditor->GetActiveViewport()->GetClient()` fallback, then `FEditorViewportClient::SetGameView(bEnabled)` (`EditorViewportClient.h:1319`, sets `bInGameViewMode` synchronously) + a synchronous `Viewport->Draw()` so a same-stack screenshot sees the icon-free frame, and report the REAL `IsInGameView()` (`:1325`); when no viewport resolves, return `VIEWPORT_NOT_AVAILABLE` instead of the phantom success. Files: `Source/PinWright/Private/Handlers/Editor/ViewportHandler.cpp`. Regression test added: `PinWright.editor.set_game_view.TogglesRealViewportState` (`Source/PinWright/Private/Tests/EditorOps/SetGameViewTogglesTest.cpp`) — resolves the same viewport client and asserts, on enable/disable, that the real `IsInGameView()` follows the request and the readback equals it (and, with no viewport, that the handler no longer reports phantom success); reverting to the exec fails it.
