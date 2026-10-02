---
id: B-screenshot-designer-preview-draws-dashed-outlines
title: "widget.screenshot_designer target:preview draws the Designer's dashed per-widget outlines into the frame, while the wiki says preview captures omit Designer chrome"
status: DONE
severity: Medium
category: bug
tags: [widget, screenshot_designer, designer, preview, chrome, outlines, docs]
encounters: 1
costly: 0
lastSeen: 2026-09-23T18:30:00Z
---

# `target:"preview"` captures carry the Designer's dashed widget outlines

Split out of `B-screenshot-designer-not-runtime-faithful` (Symptom 3). That ticket's
runtime-Visibility half is fixed. This half is a different mechanism, and fixing it is a larger
change.

**Evidence:** `Saved/Screenshots/WidgetDesigner/hud_layout_01.png` and `hud_vis_probe.png` on the
reporting host show a dashed rectangle around every widget. The page (`widget.md` /
`visual-review.md`) says preview captures "omit Designer chrome (rulers, anchor handles, selection
outlines)".

**Mechanism (source-verified, UE 5.8):** the preview UUserWidget is created with
`FWidgetBlueprintEditor::GetCurrentDesignerFlags()`. That includes `EWidgetDesignFlags::ShowOutline`
whenever `bShowDashedOutlines` is set, and that setting defaults from
`UWidgetDesignerSettings::bShowOutlines`. At design time `UWidget::TakeWidget_Private` wraps every
widget through `RebuildDesignWidget` / `CreateDesignerOutline` (`Widget.cpp:1063-1076`,
`:1141-1163`). The wrapper is an `SOverlay` holding a `MarchingAnts` `SBorder`, and the border's
visibility is fixed when the Slate widget is built. `WidgetDesignerCaptureUtil::CapturePreviewToPng`
renders `PreviewWidget->TakeWidget()`, so the wrappers come along with it. A per-capture toggle
cannot remove them. It would need a preview built without `ShowOutline` (for example
`SetShowDashedOutlines(false)` plus a preview rebuild, restored afterwards), or a render of the
unwrapped content.

**Workaround:** turn off "Show Outlines" in the Designer toolbar (the `bShowOutlines` designer
setting) before capturing, or accept the outlines as layout-probe chrome.

## History
- `#1-split-from-runtime-faithful` `OPEN` developer — Split from `B-screenshot-designer-not-runtime-faithful` #1 (Symptom 3), with the mechanism above read from engine source. Not fixed: removing a wrapper that is fixed at build time needs a preview rebuild under different designer flags. That is a bigger change than the runtime-Visibility fix landed there, and it is independent of it. Docs still claim no outlines.
- `#2-collapse-outline-borders-for-the-draw` `IN-REVIEW` developer — Took the "render without the wrappers" route rather than a preview rebuild: a rebuild under different designer flags would replace the preview `UUserWidget` and drop the transient `showOnly` / `hide` / `visibilityOverrides` already applied to it. `CapturePreviewToPng` now walks the preview's Slate tree and, for the duration of the one `FWidgetRenderer` draw, collapses every `UWidget::CreateDesignerOutline` border (an `SOverlay` carrying `FInvisibleToWidgetReflectorMetaData` whose second slot is an `SBorder` with the same metadata and is not already collapsed), restoring each border's exact prior visibility afterwards (`FPWScopedHideDesignerOutlines`). The response reports `designerOutlinesHidden`. Wiki `widget.md` now states the outlines are removed and when. Files: `Source/PinWright/Private/Handlers/UI/WidgetDesignerCaptureUtil.{h,cpp}`, `WidgetDesignerScreenshotHandler.cpp`, `docs/wiki-src/widget.md`, `CHANGELOG.md`. Test: `PinWright.widget.screenshot_designer.PreviewOmitsDesignerDashedOutlines` (forces `UWidgetDesignerSettings::bShowOutlines`, full-canvas solid fill, asserts every pixel is the fill and `DesignerOutlinesHidden >= 1`).
- `#3-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). `PinWright.widget.screenshot_designer.PreviewOmitsDesignerDashedOutlines` passed in w23-final: with `bShowOutlines` forced on, a full-canvas solid fill captures as the fill on every pixel and `designerOutlinesHidden >= 1`. The wiki claim (preview captures omit Designer chrome) now matches behaviour. `TransientOverridesRevert` and `PreviewRendersRuntimeVisibility` also passed, so collapsing the borders for the draw did not disturb transient overrides.
