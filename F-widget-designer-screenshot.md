---
id: F-widget-designer-screenshot
title: "Capture Widget Blueprint designer screenshots"
status: DONE
severity: Medium
category: feature
tags: [widget, designer, screenshot, visual-qa]
---

# Capture Widget Blueprint designer screenshots

MCP can open a Widget Blueprint with `editor.open_asset`, and it can capture the active game viewport with `editor.screenshot`, but it cannot capture the Widget Blueprint designer tab or preview area. For UMG asset work, viewport screenshots are the wrong target: the useful visual QA surface is the asset editor's Designer preview for one widget blueprint.

This blocks visual verification of asset-only widget changes such as pause-menu hotkey layout edits. Agents can inspect `widget.export_xml` and `widget.describe`, but cannot confirm spacing, overlap, clipping, or overall visual balance without a screenshot of the Designer preview.

Desired behavior:

- Open the Widget Blueprint if it is not already open.
- Focus the Designer tab, not the graph tab.
- Capture either the Designer preview area or the containing asset-editor window.
- Save a PNG to a deterministic project path and return path, width, height, and capture target metadata.
- Optionally accept a preview size or zoom setting so visual QA can be repeated at known dimensions.
- Fail with a clear error if the asset editor cannot be opened or the Designer preview cannot be located.

Suggested shape:

```json
{
  "path": "widget.screenshot_designer",
  "args": {
    "widgetPath": "/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu",
    "filename": "W_HUD_DroneGameMenu_designer.png",
    "target": "preview",
    "previewSize": { "w": 1920, "h": 1080 }
  }
}
```

The existing `editor.screenshot` implementation uses `GEngine->GameViewport` / `FScreenshotRequest`, so it cannot satisfy this feature by itself. This needs an editor-window or Slate-widget capture path aimed at the Widget Blueprint designer surface.

## History

- `#1-initial-request` `OPEN` reporter -- During montage pause-menu UMG work, the user asked to open a single Widget Blueprint and capture how it looks in the Designer. Existing MCP methods only capture the game viewport, not the Widget Blueprint editor/designer surface.
- `#2-widget-designer-screenshot` `IN-REVIEW` developer -- Added `widget.screenshot_designer` to capture the Widget Blueprint Designer preview/window via Slate, save a PNG under `Saved/Screenshots/WidgetDesigner`, return capture metadata, and added `FWidgetScreenshotDesignerRequiresWidgetPathTest` for registration/validation.
- `#3-returned-preview-not-found` `OPEN` tester — Returned: `widget.screenshot_designer` did not capture a Designer preview or window after opening the target asset. Test: `editor.open_asset assetPath:"/Game/App/UI/Test/W_McpReviewTemp_20260429.W_McpReviewTemp_20260429"` returned `success:true`, then `widget.screenshot_designer widgetPath:"/Game/App/UI/Test/W_McpReviewTemp_20260429" target:"window"` returned `PREVIEW_SLATE_NOT_FOUND`. Expected: PNG path plus width/height/capture metadata. Actual: no screenshot and no metadata.
- `#4-designer-window-fallback` `IN-REVIEW` developer -- Changed widget.screenshot_designer to avoid invalidating the preview immediately before capture, allow target:"window" to capture the asset-editor host window without requiring preview Slate, retry preview capture after Slate has a chance to attach the Designer preview, and added a regression test for window capture after editor.open_asset.
- `#5-verified-window-capture` `DONE` tester — Verified live on duplicated temp widget `/Game/McpVerify/W_McpVerifyTemp.W_McpVerifyTemp`: `widget.screenshot_designer target:"window" filename:"mcp-review-w-temp-window"` returned PNG `Saved/Screenshots/WidgetDesigner/mcp-review-w-temp-window.png` with `width:2208`, `height:1374`, `sizeBytes:343154`, `captureSource:"editorWindow"`, and `target:"window"`.
