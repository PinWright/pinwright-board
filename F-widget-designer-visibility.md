---
id: F-widget-designer-visibility
title: "Set Widget Blueprint hierarchy eye visibility"
status: DONE
severity: Medium
category: feature
tags: [widget, designer, hierarchy, visibility, editor-only]
---

# Set Widget Blueprint hierarchy eye visibility

MCP can set runtime widget `Visibility`, but it does not expose the editor-only visibility controlled by the eye icon in the Widget Blueprint Hierarchy panel. These are different states. Runtime `Visibility=Collapsed/Visible` changes the authored widget property used by gameplay, while the hierarchy eye toggles the Designer-only flag used to hide or show widgets in the editor preview.

UE stores this hierarchy eye state on `UWidget::bHiddenInDesigner`. Engine references:

- `Runtime/UMG/Public/Components/Widget.h` declares `bHiddenInDesigner`.
- `Editor/UMGEditor/Private/Hierarchy/SHierarchyViewItem.h` toggles `Item.GetTemplate()->bHiddenInDesigner = !IsVisible`.

This is needed for visual QA of dense Widget Blueprints. For example, when editing one pause-menu hotkey layer, an agent needs to hide sibling layers in the Designer without changing their runtime `Visibility` defaults or activation logic.

Desired behavior:

- Set editor-only Designer visibility for a named widget-tree child.
- Do not modify runtime `Visibility`.
- Update the template widget and, if the Widget Blueprint editor is open, refresh the preview instance so the eye icon and Designer surface reflect the change.
- Optionally support recursive children behavior if that matches Designer UX.
- Return old/new state and whether the asset was dirtied or only editor-session state changed.

Suggested shape:

```json
{
  "path": "widget.set_designer_visibility",
  "args": {
    "widgetPath": "/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu",
    "widgetName": "HorizontalBox-Montage",
    "visible": true
  }
}
```

Possible complementary read API:

```json
{
  "path": "widget.get_designer_visibility",
  "args": {
    "widgetPath": "/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu",
    "widgetName": "HorizontalBox-Montage"
  }
}
```

`widget.get_class_properties` currently does not surface `bHiddenInDesigner`, so callers should not have to rely on generic `widget.set` or raw property access for this editor-only flag.

## History

- `#1-initial-request` `OPEN` reporter -- During montage pause-menu UMG work, the user asked whether MCP can toggle the Widget Blueprint hierarchy eye icon separately from runtime `Visibility`. Existing `widget.set` covers runtime visibility only; MCP needs a dedicated API for `UWidget::bHiddenInDesigner`.
- `#2-designer-visibility-api` `IN-REVIEW` developer -- Added dedicated `widget.get_designer_visibility` and `widget.set_designer_visibility` handlers for `UWidget::bHiddenInDesigner`, preserving runtime `Visibility`, updating open Widget Blueprint preview state, and covering the behavior with `FWidgetDesignerVisibilitySetDoesNotChangeRuntimeVisibilityTest`.
- `#3-verified-designer-visibility` `DONE` tester — Verified: `widget.get_designer_visibility` on `/Game/App/UI/Test/W_McpReviewTemp_20260429` `AlphaText` returned `visible:true`, `hiddenInDesigner:false`, `visibility:"Visible"`; `widget.set_designer_visibility visible:false` returned `oldVisible:true`, `newVisible:false`, `runtimeVisibilityUnchanged:true`; a second get returned `visible:false`, `hiddenInDesigner:true`, while runtime `visibility` stayed `Visible`.
