---
id: F-widget-screenshot-designer-resolution
title: "widget.screenshot_designer needs resolution/scale parameter"
status: DONE
severity: Medium
category: feature
tags: [widget, screenshot, designer, resolution]
---

# widget.screenshot_designer needs resolution/scale parameter

`widget.screenshot_designer` with `target="preview"` returns a PNG sized to whatever the live UMG Designer preview canvas currently measures on screen. Real-world capture from `/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu` came out 671x378 — too low-resolution for visual diffing, design review, or any downstream image-processing use case.

The original handler called `FSlateApplication::TakeScreenshot` against the resolved `SDesignerView` widget with a preview crop rect, so output dimensions were determined by the live Slate back-buffer (editor layout, panel widths, monitor DPI, the asset's "Fill Screen / Custom" Designer size), none of which the caller controls through the RPC.

## What's needed

A single integer cap on the longer axis of the output image. The shorter axis is derived from the asset's natural aspect ratio (preview geometry's absolute size).

- `max_size` (integer, optional, default `1024`) — maximum pixel size on the longer output axis. The other axis is computed as `round(naturalShort * (max_size / naturalLong))`. Hard-capped at 16384 to prevent unbounded render-target allocations.
- Applies only to `target="preview"`. Ignored for `target="window"` (the editor host window is captured at its live back-buffer size, which is what callers want for window mode).

Output dimensions are now caller-controlled rather than monitor-DPI-bound.

## Why this matters

- **Visual diffing / regression tests** need stable, high-res output independent of the developer's monitor and editor layout.
- **Design review screenshots** at 671x378 are unreadable when widgets contain text or fine detail.
- **Documentation captures** for the wiki are too small to be useful at native resolution.

## Workaround (pre-fix)

Manually resize the WBP's Designer canvas (Custom size in the Designer toolbar) before capturing — but this dirtied the asset, required an explicit revert, and the size is a UPROPERTY that persisted if saved by accident. `target="window"` captures more pixels but includes editor chrome.

## Fix

Switch the `target="preview"` capture path from `FSlateApplication::TakeScreenshot` (live back-buffer readback against `SDesignerView`) to `FWidgetRenderer::DrawWidget` rendering the preview `UUserWidget`'s Slate widget into a sized `UTextureRenderTarget2D`, then `FRenderTarget::ReadPixels` into the existing PNG-encode path.

- The render-target's draw size is `(round(natural.X * max_size / longAxis), round(natural.Y * max_size / longAxis))`, where `natural = DesignerTarget.PreviewGeometry.GetAbsoluteSize()`.
- `FlushRenderingCommands` is required between `DrawWidget` and `ReadPixels` — the draw is enqueued on the render thread, and reading without flushing returns black pixels.
- The visible output no longer includes Designer chrome (rulers, anchor handles, selection outlines). It is closer to a runtime-style capture of the preview widget content. This is intentional and matches the use cases (visual diffing, design review, documentation captures) but is a behavioral change vs. the old crop-from-SDesignerView path.
- `target="window"` is unchanged — it still uses `FSlateApplication::TakeScreenshot` against the editor host window.

The response payload gains `max_size` (integer, echo of the applied cap) and `renderer` (`"widgetRenderer"` for preview, `"slateScreenshot"` for window). The existing `width`/`height` fields now report the actual output dimensions, which are the caller-controlled `DrawSize` for preview captures.

## History
- `#1-initial-request` `OPEN` reporter — User asked whether `widget.screenshot_designer` can produce higher-res captures. Wiki shows no resolution param; handler is bound to live Designer canvas geometry (671x378 measured on `W_HUD_DroneGameMenu`). Filed for an optional `scale` or `width/height` param driving `FWidgetRenderer` target size.
- `#2-max-size-and-widgetrenderer-swap` `IN-REVIEW` developer — Added `max_size` integer parameter (default 1024) capping the longer output axis with aspect preserved. Switched `target=preview` capture path from `FSlateApplication::TakeScreenshot` to `FWidgetRenderer::DrawWidget` into a sized `UTextureRenderTarget2D` so output resolution is caller-controlled rather than monitor-DPI-bound. `target=window` path unchanged. Added `FWidgetScreenshotDesignerMaxSizePreviewTest` covering landscape, portrait, and default-size cases.
- `#3-rt-lifecycle-and-cleanup` `IN-REVIEW` developer — Rooted the transient `UTextureRenderTarget2D` via `TStrongObjectPtr` (or `AddToRoot`/`RemoveFromRoot`) for the duration of the capture so a GC tick during `FlushRenderingCommands` cannot collect it mid-readback, and `ReleaseResource()`/`MarkAsGarbage()` on exit so GPU memory frees promptly. Corrected the misleading "4× 4K" comment on the `max_size` cap.
- `#4-verify-fix` `DONE` tester — Verified: `widget.screenshot_designer` wiki schema exposes `max_size` (integer, optional, default 1024). Live call on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu` with `target=preview, max_size=2048` returned `width=2048, height=1152` (16:9 aspect preserved), `renderer="widgetRenderer"`, `max_size=2048` echoed — caller-controlled output replacing the previous 671x378 monitor-DPI-bound capture.
