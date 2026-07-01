---
id: B-widget-screenshot-preview-includes-chrome
title: "widget.screenshot_designer target:preview can fail with PREVIEW_SLATE_NOT_FOUND from cold Designer state"
status: DONE
severity: Medium
category: bug
tags: [widget, designer, screenshot, preview-lifecycle]
---

# widget.screenshot_designer target:preview can fail with PREVIEW_SLATE_NOT_FOUND from cold Designer state

`widget.screenshot_designer` with `target: "preview"` is a one-call preview capture API: callers pass `widgetPath` and should receive a `captureSource: "designerPreview"` PNG without first opening or ticking the Widget Blueprint Designer manually.

The current crop behavior is already correct: `target: "preview"` captures from the Designer view anchor and crops to the preview canvas/root geometry, so it no longer includes Blueprint editor chrome. The active failure is lifecycle-related. From a cold Designer state, opening the Widget Blueprint can leave the preview `UUserWidget` or its cached Slate widget unwarmed when the handler immediately resolves `target: "preview"`, so the call can fail with `PREVIEW_SLATE_NOT_FOUND` even though the editor can show the same preview after a manual open/tick.

## Repro

1. Start from a cold editor state where the Widget Blueprint is not already open in the Designer tab.
2. Call `widget.screenshot_designer widgetPath:"/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu" filename:"mcp_verify_preview_canvas_crop.png" target:"preview"`.
3. The handler opens the asset but can return `PREVIEW_SLATE_NOT_FOUND: Widget Blueprint Designer target not available` instead of a preview PNG.

## Expected

Preview-mode capture self-opens the Widget Blueprint, switches to the Designer, invokes the Slate preview tab, warms the preview Slate/bounds with bounded retries, and returns a PNG cropped to the rendered preview canvas/root. If the preview Slate or crop bounds still cannot be resolved after retries, the handler should fail explicitly with `PREVIEW_SLATE_NOT_FOUND` or `PREVIEW_BOUNDS_NOT_FOUND`.

## Analysis

The previous canvas-crop fix is the right capture boundary. The remaining bug is that the handler resolves the cached preview Slate too early after opening the editor. `FWidgetBlueprintEditor::RefreshPreview()` can recreate the preview `UUserWidget`, while the Designer view and `GetCachedWidget()` need Slate tab creation and paint/layout ticks before `FWidgetGeometryResolver::ResolveDesignerPreviewTarget()` can resolve the preview Slate and crop rectangle.

The fix should keep `target: "preview"` strict: warm the real Designer preview and crop bounds, but never fall back to the editor window or whole `SDesignerView` when preview resolution fails. `target: "window"` remains the explicit chrome-including mode.

## Workaround

Open the Widget Blueprint in the Designer, wait for the preview to paint, then retry the screenshot. This defeats the one-call screenshot contract and is not suitable for automated visual QA.

## Relation to existing items

- `F-widget-designer-screenshot.md` (DONE) introduced the handler and verified `target:"window"`.
- Prior entries on this ticket fixed the preview crop boundary; this returned scope is only the cold Designer preview lifecycle failure.

## History

- `#1-initial-report` `OPEN` reporter — During montage HUD visual QA, an agent captured `W_HUD_DroneGameMenu` with `target:"preview"`. The PNG includes the Widget Blueprint editor's Hierarchy panel (left) and Details panel (right), with the actual preview taking only the centre. Response advertised `captureSource:"designerPreview"` so this is a capture-bounds bug, not a misuse — the handler claims preview but delivers windowed content.
- `#2-capture-sdesignerview-not-userwidget` `IN-REVIEW` developer — Switched preview target to SDesignerView via tab lookup + parent-walk fallback. Added DesignerSurfaceSlate field on FWidgetDesignerPreviewTarget; populated in WidgetGeometryResolver::ResolveDesignerPreviewTarget. Handler uses the surface slate for target:"preview" (PREVIEW_NOT_FOUND on failure, no silent fallback). Added FWidgetScreenshotDesignerPreviewExcludesEditorChromeTest comparing capture width/height against the editor host window.
- `#3-preview-still-chrome` `OPEN` tester — Returned: `widget.screenshot_designer` for `/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu` with `target:"preview"` saved `mcp_verify_preview_chrome.png` as `captureSource:"designerPreview"` at 1118x1112, but the PNG still includes designer rulers, zoom/lock/grid toolbar, screen-size controls, and Width/Height inputs instead of only the rendered widget/design canvas. Test: `widget.screenshot_designer widgetPath:"/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu" filename:"mcp_verify_preview_chrome.png" target:"preview"`.
- `#4-preview-canvas-crop` `IN-REVIEW` developer — Changed preview screenshots to use the designer view only as the anchor and crop to the resolved preview canvas/root geometry instead of capturing the whole SDesignerView; added preview-bounds regression coverage.
- `#5-regression-preview-slate-not-found` `OPEN` tester — Returned: `widget.screenshot_designer` for `/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu` with `target:"preview"` now fails with `PREVIEW_SLATE_NOT_FOUND: Widget Blueprint Designer target not available`, even though `editor.open_asset` succeeds and the same widget captures fine with `target:"window"` (2216x1343 PNG showing the open Designer with rendered preview). The new canvas-crop path appears to have broken preview-target resolution: previous IN-REVIEW (#3) at least produced a chrome-leaking PNG; this revision produces no PNG at all. Test: `editor.open_asset assetPath:"/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu"` then `widget.screenshot_designer widgetPath:"/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu" filename:"mcp_verify_preview_canvas_crop.png" target:"preview"` (re-opened the asset and retried, same error).
- `#6-handler-should-self-open-designer` `OPEN` reporter — Expand scope: `widget.screenshot_designer` should not require a separate `editor.open_asset` precondition. Caller passes `widgetPath`; the handler is the right place to ensure the Widget Blueprint is opened in the Designer tab, take the screenshot, and (ideally) restore the prior editor tab state. Today the verifier had to call `editor.open_asset` first or get `PREVIEW_SLATE_NOT_FOUND`, which defeats the "one-call screenshot" contract callers expect. Fix should be folded into the `target:"preview"` rework so a cold call from any editor state produces a valid preview PNG. `target:"window"` should follow the same self-open behavior for consistency.
- `#7-preview-self-open-warmup` `IN-REVIEW` developer — Reformulated the ticket around the active cold Designer preview lifecycle failure, warmed the Widget Blueprint Designer preview before resolving `target:"preview"` crop bounds, and added `FWidgetScreenshotDesignerPreviewSelfOpensDesignerTest` so a cold call fails if it still returns `PREVIEW_SLATE_NOT_FOUND`.
- `#8-returned-preview-slate-not-found` `OPEN` tester — Returned: `widget.screenshot_designer` for `/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu` with `target:"preview"` still fails with `PREVIEW_SLATE_NOT_FOUND: Widget Blueprint Designer target not available`, so the cold one-call preview lifecycle fix is not working. Test: `widget.screenshot_designer widgetPath:"/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu" filename:"mcp_verify_preview_self_open_warmup.png" target:"preview"`.
- `#9-designerview-mixin-and-tick-race` `IN-REVIEW` developer — Two compounding defects in the cold preview lifecycle: ResolveDesignerViewWidget's strict GetType()=='SDesignerView' predicate never matched UE 5.6's TToolCompatibleMixin<SDesignerView> wrapper, and the warmup retry loop called RefreshPreview() on every attempt which re-destroyed the UUserWidget before SDesignerView::Tick could call TakeWidget(). Predicate now substring-matches 'SDesignerView'; PREVIEW_SLATE_NOT_FOUND vs DESIGNER_SURFACE_NOT_FOUND now distinguished; retry loop only pumps Slate (PumpMessages+Tick+ForceRedraw) and never rebuilds the preview; MaxSlateResolveAttempts=12 with 5ms sleeps; idle throttle woken once via host-window BringToFront. FWidgetScreenshotDesignerPreviewSelfOpensDesignerTest strengthened with explicit cold-state preconditions and a non-empty preview canvas.
- `#10-verify-cold-preview-capture` `DONE` tester — Verified: cold one-call `widget.screenshot_designer widgetPath:"/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu" filename:"mcp_verify_designerview_mixin_fix.png" target:"preview"` with no prior `editor.open_asset` returned `captureSource:"designerPreview"` at 671x378, `previewName:"W_HUD_DroneGameMenu_C_0"`. PNG shows only the rendered widget preview canvas — no Hierarchy/Details panels, no rulers, no zoom toolbar, no Width/Height inputs. Lifecycle warmup fix holds.
