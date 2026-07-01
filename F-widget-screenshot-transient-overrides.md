---
id: F-widget-screenshot-transient-overrides
title: "widget.screenshot_designer should accept transient designer-eye and visibility overrides"
status: DONE
severity: Medium
category: feature
tags: [widget, designer, screenshot, visibility, ergonomics]
---

# widget.screenshot_designer should accept transient designer-eye and visibility overrides

`widget.screenshot_designer` captures the current Designer state of a Widget Blueprint, but offers no way to scope what's shown for the screenshot. To capture a single subtree (e.g. one mode-specific overlay inside a multi-mode pause HUD), the caller has to either:

1. Permanently flip sibling widgets' `bHiddenInDesigner` via `widget.set_designer_visibility`, screenshot, then flip them back — every mutation marks the asset dirty and risks accidental save persisting the test state.
2. Flip sibling runtime `Visibility=Collapsed` via `widget.set` — wrong tool, mutates real game behavior, persists on save.
3. Manually arrange the Designer state in the editor before each capture — defeats the point of an MCP-driven screenshot loop.

In practice both (1) and (2) require remembering to revert before the next save_all call. For visual-QA loops (capture → diff → tweak → capture again), the dirty-revert dance is tedious and error-prone, and one mistake bakes the test-only state into the asset.

## Desired behavior

`widget.screenshot_designer` accepts optional transient overrides applied for the duration of the capture only — never marking the asset dirty, never persisting:

- `showOnly: ["WidgetName1", "WidgetName2"]` — designer-eye-show only the named widgets and their ancestors; designer-eye-hide every other sibling under the shared root. After capture, restore prior `bHiddenInDesigner` state on every touched widget.
- `hide: ["WidgetName3"]` — designer-eye-hide the named widgets in addition to their saved state.
- `visibilityOverrides: { "WidgetName1": "Visible", "WidgetName2": "Collapsed" }` — temporarily swap the runtime `Visibility` UPROPERTY on the named widgets *for the rendered preview only*, restoring before save. This is needed because the Designer preview honors runtime `Visibility=Collapsed` for layout — designer-eye alone won't make a Collapsed subtree render.

All overrides are reverted in a `try/finally`-equivalent block so a render failure or RPC error still restores the original state.

## Suggested shape

```json
{
  "path": "widget.screenshot_designer",
  "args": {
    "widgetPath": "/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu",
    "filename": "W_HUD_DroneGameMenu_Montage.png",
    "target": "preview",
    "showOnly": ["HorizontalBox-Montage"],
    "visibilityOverrides": { "HorizontalBox-Montage": "Visible" }
  }
}
```

After capture: every overridden widget's `bHiddenInDesigner` and `Visibility` are reverted to their saved values; asset is not dirty unless other operations dirtied it.

## Workaround

Today: pair `widget.set_designer_visibility` (per widget, dirties asset) with `widget.set` for runtime Visibility (dirties asset, real semantic change), capture via `widget.screenshot_designer`, then revert both — and never call `editor.save_all` between override and revert. Brittle.

## History

- `#1-initial-request` `OPEN` reporter — During visual QA of a mode-specific Help overlay (`HorizontalBox-Montage` inside `W_HUD_DroneGameMenu`), the user wanted a screenshot showing only that subtree. The current API forced permanent designer-eye flips on three sibling overlays plus a runtime-Visibility flip on the target box, all of which mark the asset dirty and persist on the next save.
- `#2-transient-overrides-implemented` `IN-REVIEW` developer — Added `showOnly`, `hide`, `visibilityOverrides` optional params to `widget.screenshot_designer`. Override apply/revert is encapsulated in `WidgetAuthoringHelpers::ApplyTransientDesignerOverrides` / `RevertTransientDesignerOverrides` (declared in `WidgetAuthoringUtils.h`). Mutations land on the live preview UUserWidget only — the asset package is never marked dirty. `ON_SCOPE_EXIT` guarantees revert on success, error, or exception. Regression test `FWidgetScreenshotDesignerTransientOverridesRevertTest` proves the preview eye flag returns to its pre-capture value and the asset dirty flag is unchanged. Wiki overlay at `docs/wiki/widget.md` extended with an `#### Override modes` H4 inside the `### widget.screenshot_designer` H3.
- `#3-verify-transient-overrides` `DONE` tester — Verified: `call("widget.screenshot_designer")` documents `showOnly`, `hide`, and `visibilityOverrides`; `asset.search` found `/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu.W_HUD_DroneGameMenu`; live capture with `showOnly: ["HorizontalBox-Montage"]` and `visibilityOverrides: {"HorizontalBox-Montage":"Visible"}` wrote `McpVerify_F_widget_screenshot_transient_overrides.png` with `width: 1118`, `height: 1112`, `sizeBytes: 85329`, and `overridesApplied` showing `hiddenOverrideCount: 99`, `visibilityOverrideCount: 1`, `showOnlyInputCount: 1`.
