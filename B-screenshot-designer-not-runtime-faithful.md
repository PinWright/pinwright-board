---
id: B-screenshot-designer-not-runtime-faithful
title: "widget.screenshot_designer target:preview renders designer-decorated state — runtime Visibility=Collapsed still draws, visibilityOverrides is a silent no-op, and every widget gets a dashed outline"
status: OPEN
severity: Medium
category: bug
tags: [widget, screenshot_designer, designer, preview, visibility, visibilityOverrides, chrome, silent-noop]
encounters: 1
lastSeen: 2026-09-02T19:35:00Z
---

# `widget.screenshot_designer` `target:"preview"` is not a runtime-faithful render

Three symptoms, one likely root cause: the `target:"preview"` render is the **Designer's decorated
preview**, not the runtime widget. The wiki page claims the opposite twice, and one advertised
parameter reports success while doing nothing.

## Symptom 1 — runtime `Visibility=Collapsed` widgets are rendered

`/Game/FPS/UI/WBP_HUD` has `ReloadText`, `PromptRoot` and `FeedRow0..3` authored with
`Visibility="Collapsed"`. `widget.describe` confirms it on the asset:

```
widget.describe {widgetPath:"/Game/FPS/UI/WBP_HUD", widgetName:"ReloadText"}
  -> props.Visibility = "Collapsed"
```

The preview capture draws all of them at full opacity anyway
(`Saved/Screenshots/WidgetDesigner/hud_layout_01.png` — "RELOADING", "F HOLD TO OPEN" and four
kill-feed rows are all visible).

The wiki page states the opposite, under "Runtime Visibility=Collapsed leaks into the preview
render": *"The Designer honors runtime `Visibility=Collapsed`: a Collapsed parent contributes zero
size and children do not render even when their eye flag is shown."* On this build it does not.

## Symptom 2 — `visibilityOverrides` is a silent no-op in the hide direction

```
widget.screenshot_designer {
  widgetPath: "/Game/FPS/UI/WBP_HUD", target: "preview", max_size: 1200,
  visibilityOverrides: { "ReloadText": "Collapsed", "PromptRoot": "Collapsed" },
  hide: ["FeedRow0", "FeedRow1"]
}
-> { ..., overridesApplied: { hiddenOverrideCount: 2, visibilityOverrideCount: 2,
                              showOnlyInputCount: 0 } }
```

In the resulting PNG (`Saved/Screenshots/WidgetDesigner/hud_vis_probe.png`):

- `hide` **works** — `FeedRow0` / `FeedRow1` are gone (two kill-feed rows remain).
- `visibilityOverrides` **does nothing** — "RELOADING" and "F HOLD TO OPEN" still render, even
  though the response reports `visibilityOverrideCount: 2`.

So the response asserts an override was applied that the pixels disprove. Both directions are
broken consistently with Symptom 1: runtime `Visibility` simply does not reach this render path.
`visibilityOverrides` was added by `F-widget-screenshot-transient-overrides` (DONE) specifically
because "the Designer preview honors runtime `Visibility=Collapsed`" — that premise does not hold
here, which is presumably why the parameter has no observable effect either way.

## Symptom 3 — every widget is drawn with a dashed outline

Both captures show a dashed rectangle around **every** widget in the tree — text blocks, boxes,
canvases. The wiki page says: *"Preview captures use `FWidgetRenderer::DrawWidget`, not the live
back buffer, so they omit Designer chrome (rulers, anchor handles, selection outlines) and resemble
runtime widget content."* Dashed per-widget outlines are Designer chrome and they are in the frame.
This is distinct from `B-widget-screenshot-preview-includes-chrome` (DONE), which was about the
crop including *editor window* chrome; the crop here is correct and the decoration is inside it.

## Root-cause guess (not source-verified)

All three follow if the handler renders the Designer's preview widget **with the designer wrapper
still installed** (an `SDesignerWidget`-style decorator per child that forces visibility so a
hidden widget stays selectable, and draws the dashed bound) rather than a clean
`UUserWidget::TakeWidget()` of the preview instance. That would explain visible Collapsed widgets,
inert visibility overrides, and dashed outlines from a single cause. I did not read the handler
source, so this is a hypothesis.

## What should happen

`target:"preview"` should render the widget as it would appear at runtime: honour runtime
`Visibility`, honour `visibilityOverrides` in both directions, and draw no per-widget outlines —
that is what the page promises and what makes the capture usable as visual evidence. If the
decorated render is deliberate, the page must say so and `visibilityOverrides` must either work or
be rejected rather than reported as applied.

**Workaround:** use `hide` / `showOnly` (designer-eye), which do work, and treat the preview capture
as a layout probe only — verify final appearance in PIE with `editor.screenshot`.

severity rationale: impact=silent false-success (a reported override that the pixels disprove) plus
a wiki page that documents the inverse behaviour, forcing a wrong mental model x reach=widget
preview capture is the standard UMG review loop -> Medium (no data loss, and `hide` is a working
workaround, so not High).

## History
- `#1-filed` `OPEN` reporter — Hit while building `/Game/FPS/UI/WBP_HUD` on EAContentExamples58 (UE 5.8). Authored six widgets with `Visibility="Collapsed"` through `widget.import_xml`; `widget.describe` confirms `Visibility: "Collapsed"` on the asset, but `widget.screenshot_designer target:"preview"` draws them. Ran a controlled probe in one call with `hide:["FeedRow0","FeedRow1"]` and `visibilityOverrides:{"ReloadText":"Collapsed","PromptRoot":"Collapsed"}`: the response reported `hiddenOverrideCount:2, visibilityOverrideCount:2`, the two `hide` targets vanished from the PNG, and neither `visibilityOverrides` target changed — so the eye path works and the runtime-Visibility path is inert, in both the authored and the override direction. Evidence PNGs: `Saved/Screenshots/WidgetDesigner/hud_layout_01.png` (1920x1080) and `Saved/Screenshots/WidgetDesigner/hud_vis_probe.png` (1200x675). Both also show a dashed outline around every widget, which the same wiki page says preview captures omit. Cross-refs: `F-widget-screenshot-transient-overrides` (DONE) added `visibilityOverrides` on the premise that the preview honours runtime Collapsed — that premise is false on this build; `E-widget-screenshot-docs-eye-vs-visibility` (DONE) wrote the wiki paragraph that is now inverted; `B-widget-screenshot-preview-includes-chrome` (DONE) covered window-chrome cropping, not per-widget outlines. Root cause not source-verified.
