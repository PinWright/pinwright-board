---
id: B-screenshot-omits-umg-overlay
title: "ui.screenshot / editor.screenshot capture only the 3D scene render target — the live UMG/Slate viewport overlay (AddToViewport HUD) is silently absent from the PNG"
status: IN-REVIEW
severity: High
category: bug
tags: [ui, screenshot, editor, umg, slate, hud, pie, readpixels, silent-wrong, visual-review]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# Screenshot capture omits the live UMG/Slate overlay (HUD added via AddToViewport)

`ui.screenshot` (and the canonical `editor.screenshot`, which `B-editor-screenshot-no-completion-signal #6` reworked to use the **same** capture mechanism) capture the PIE/game viewport by reading
`GEngine->GameViewport->Viewport->ReadPixels(Bitmap)`
(`Source/.../Utils/ScreenshotUtils.cpp` `CaptureGameViewportToPngFile`, L66-78). That call reads the viewport's **scene render-target back buffer** only. UMG widgets added to the viewport at runtime via `AddToViewport` (`ui.create_hud`, `ui.activatable_push`, or any `CreateWidget`+`AddToViewport`) are composited by Slate at the **window** layer, on top of the viewport's render target — they are NOT in the buffer `FViewport::ReadPixels` returns. The result: a screenshot taken during PIE shows the 3D scene but **silently omits the entire live HUD**.

This is a **silent wrong-data** result on a normal, high-traffic path: the call returns a clean success with valid metadata (`width`, `height`, `path`, `sizeBytes`) and a real PNG of the scene, so a caller who asked for a screenshot "to eyeball the HUD layout" receives — and trusts — a frame that misrepresents what is actually on screen. There is no error, no warning, nothing that signals the UI layer is missing. The caller builds on a lie. For HUD/widget visual review (the obvious reason to screenshot during PIE), the tool delivers a frame with the one thing the caller wanted to see removed.

The live widgets ARE rendering — this is purely a capture-path defect, not a HUD that failed to instantiate. `widget.describe capture_source=live` on the same instance reports the driven state present and visible; the screenshot just doesn't include it.

**Impact:** any HUD-driving or widget-visual-review task during PIE. The MCP exposes a rich runtime-UI surface (`ui.create_hud`, `ui.set_widget_text/_image/_visibility`, `ui.activatable_*`) whose only visual-verification verb is the screenshot — and that verb cannot see any of it. Agents fall back to `widget.describe capture_source=live` to confirm UI state, which proves the data but never the *layout/appearance* the screenshot was meant to provide.

**Workaround:** none through this verb for UI capture. Read back UMG state structurally via `widget.describe capture_source=live` (confirms text/visibility/brush, NOT pixel layout). A real UI-inclusive capture needs the Slate-window / high-res-shot-with-UI path, not viewport `ReadPixels`.

**Fix:** capture a UI-inclusive frame when a game/PIE viewport is active. Options, in order of fidelity:
1. Use the engine's screenshot-with-UI path — `FScreenshotRequest::RequestScreenshot(Filename, /*bShowUI=*/true, ...)` and read the resulting frame, which Slate composites with the window's widget overlay. (Note `B-editor-screenshot-no-completion-signal` moved *off* the async `FScreenshotRequest` path specifically because `OnScreenshotCaptured` hung under PIE-in-editor — so a naive revert is not acceptable; a UI-inclusive sync path is needed, or the async path must be made to complete reliably.)
2. Alternatively composite the game viewport's Slate window (`SGameLayerManager` / the viewport widget) to a render target and read that, so the `AddToViewport` overlay is included.
Whichever path is chosen, the capture must include the live UMG overlay so a screenshot taken during PIE reflects what is actually on screen. If a UI-free scene capture is also wanted, expose it as an explicit option rather than as the silent default.

## Repro (verbatim, replay-confirmed)

1. `editor.play {}` → `{success:true}`
2. `ui.create_hud {widgetPath:"/Game/UI/WBP_ActionHUD"}` → `{widgetName:"WBP_ActionHUD_C_0"}`
3. `ui.set_widget_text {key:"StatusText", value:"WAVE 3 - 1200 pts"}` → `{key:"StatusText", value:"WAVE 3 - 1200 pts", owning_user_widget:"WBP_ActionHUD_C_0"}`
4. `ui.screenshot {filename:"oracle_hud_composite_check", returnBase64:false}` → `{screenshotPath:".../oracle_hud_composite_check.png", width:957, height:525, sizeBytes:15765}` — clean success.
5. Read the PNG: it shows the DemoRoom 3D scene (dark room, light shafts, a level-baked text panel) with **no** `StatusText`, **no** `StatusIcon`, **no** HUD panels — the live HUD is entirely absent.
6. `widget.describe {instance_name:"WBP_ActionHUD", capture_source:"live"}` on the SAME running instance reports `StatusText` with `"Text": "WAVE 3 - 1200 pts"` and the root `visibility: SelfHitTestInvisible` (rendering) — proving the HUD is live and on-screen, only the screenshot omits it.

## History
- `#2-ui-inclusive-capture` `IN-REVIEW` developer — Fixed the shared capture path so it composites the live Slate/UMG viewport overlay instead of reading the scene-only render target. `Source/PinWright/Private/Utils/ScreenshotUtils.cpp` `CaptureGameViewportToPngFile` now captures the game-viewport widget's window back buffer via `FSlateApplication::TakeScreenshot(GameViewportClient->GetGameViewportWidget(), ...)` — the same synchronous back-buffer mechanism `widget.screenshot_designer`'s window path already uses, so the `AddToViewport` HUD is included and there is no async `FScreenshotRequest` hang (avoids re-breaking `B-editor-screenshot-no-completion-signal #6`). Falls back to the prior `FViewport::ReadPixels` scene-only capture when Slate/back buffer is unavailable (e.g. `-RenderOffScreen` headless), and forces opaque alpha so back-buffer transparency doesn't bleed into the PNG. Both `ui.screenshot` and `editor.screenshot`'s PIE branch route through this helper, so both are fixed; `DriveSetOfMarkRenderer` uses its own inline readback and is untouched. Header doc comment updated (`Utils/ScreenshotUtils.h`). Regression test added: `Source/PinWright/Private/Tests/EditorOps/TestUiScreenshotUmgOverlay.cpp` (`PinWright.ui.screenshot.IncludesViewportUmgOverlay`) pushes a full-viewport opaque-green Slate overlay via `AddViewportWidgetContent`, calls `CaptureGameViewportToPngFile`, decodes the PNG and asserts the frame reads as the overlay color — fails on the reverted scene-only path; live-viewport-guarded (skips when no game/PIE viewport is bound, matching the sibling screenshot regression).
- `#1-initial-repro` `OPEN` reporter — Found during a live-HUD PIE preview task (author + instantiate a HUD, drive its state, screenshot to eyeball the layout). Replay-confirmed: started PIE, `ui.create_hud` of `/Game/UI/WBP_ActionHUD` (ref `WBP_ActionHUD_C_0`), set `StatusText="WAVE 3 - 1200 pts"` live, then `ui.screenshot` → clean success (957x525, 15765B) but the saved PNG is the 3D scene only — no HUD text/icon/panels. The same live instance's `widget.describe capture_source=live` confirms `StatusText.Text="WAVE 3 - 1200 pts"` and a visible (`SelfHitTestInvisible`) root, so the HUD is genuinely rendered and the screenshot capture path simply drops the Slate/UMG overlay. Root cause: `CaptureGameViewportToPngFile` reads `GEngine->GameViewport->Viewport->ReadPixels` (`ScreenshotUtils.cpp` L66-78) = scene render target only, never the window-level `AddToViewport` overlay. Distinct from the other open screenshot tickets (async hang `B-editor-screenshot-no-completion-signal`; doubled `.png.png` `E-ui-screenshot-doubles-png-extension`; JPEG-in-png `B-thumbnail-png-writes-jpeg`) — those are completion/filename/encoding defects; this is the captured *content* silently missing the UI layer. Fix: capture a UI-inclusive frame (screenshot-with-UI / composite the Slate window), not viewport `ReadPixels`, when the game/PIE viewport is active.
