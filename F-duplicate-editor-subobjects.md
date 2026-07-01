---
id: F-duplicate-editor-subobjects
title: "Duplicate widget-tree children and actor components"
status: DONE
severity: Medium
category: feature
tags: [widget-tree, actor-components, duplicate, ergonomics]
---

# Duplicate widget-tree children and actor components

## Problem

EditorAutomation MCP can add, set, rename, reparent, replace, wrap, export, and import widget data, but it does not expose a direct way to duplicate an existing widget-tree child in place. It also lacks an equivalent workflow for duplicating actor components on actors or actor Blueprints.

This forces callers to recreate authored UI or component structure manually by reading the source object, adding a new object, copying properties, copying slot or attachment state, and recreating descendants. That is slow and error-prone when the goal is to preserve exact existing layout, style, and component setup while changing only names or a few labels.

The current montage pause hotkey work hit this directly: the desired change is to duplicate or repurpose existing hotkey rows so the new montage hotkey layout keeps the exact existing text layout and styles. Without a duplicate primitive, the fallback is manual `widget.add` plus `widget.set` reconstruction from existing widget data.

## Desired Behavior

Add a first-class MCP operation for duplicating widget-tree children:

- Duplicate a named source widget under the same parent by default.
- Allow a target parent and placement, such as after source, end of parent, or explicit index.
- Preserve widget class, editable properties, slot properties, variable/export settings where safe, and nested children by default.
- Allow a required or generated new name to avoid collisions.
- Return a source-to-target name mapping for the duplicated subtree.
- Report whether compile/save is required and any properties or bindings that could not be copied safely.

Suggested shape:

```json
{
  "path": "widget.duplicate",
  "args": {
    "widgetPath": "/Game/App/UI/W_PauseMenu",
    "sourceName": "HotkeyRow_Existing",
    "newName": "HotkeyRow_MontageScroll",
    "parentName": "HotkeyContainer",
    "placement": { "after": "HotkeyRow_Existing" },
    "duplicateChildren": true,
    "copyProperties": true,
    "copySlot": true
  }
}
```

Add an equivalent component duplication workflow for actors and/or actor Blueprints:

- Duplicate a named source component on the same actor or Blueprint by default.
- Preserve component class, editable properties, attachment parent/socket, relative transform, tags, and optionally child components.
- Allow target actor/Blueprint, target parent component, placement/order where applicable, and collision-safe naming.
- Return source-to-target component mapping, compile/save requirements, and warnings for references that cannot be copied safely.

Suggested shape:

```json
{
  "path": "component.duplicate",
  "args": {
    "assetPath": "/Game/App/Blueprints/BP_SomeActor",
    "sourceName": "ExistingComponent",
    "newName": "ExistingComponent_Copy",
    "targetParentName": "RootComponent",
    "duplicateChildren": true,
    "copyProperties": true,
    "copyAttachment": true
  }
}
```

Exact namespace can follow the plugin's existing actor and Blueprint API boundaries. The important requirement is that callers can clone editor-authored subobjects directly instead of reconstructing them manually.

## Workarounds

- Widgets: inspect or export the source widget, then recreate it with `widget.add`, `widget.set`, `widget.rename_widget`, and `widget.reparent_widget`.
- Components: add a new component, copy properties manually, then recreate attachment and transform state.

These workarounds are fragile for exact UI style/layout matching and for component graphs with nested attachments.

## History

- `#1-initial-request` `OPEN` reporter -- Session needed exact duplication or repurposing of pause-menu hotkey rows for montage editor work. MCP had no direct widget-tree duplicate operation, and the user explicitly requested a board task covering widget-tree duplication and actor component duplication.
- `#2-duplicate-editor-subobjects` `IN-REVIEW` developer — Added widget.duplicate, blueprint.scs.duplicate_component, and actor.duplicate_component handlers to clone authored subobjects with mapping/warnings, plus regression tests for widget subtree slot placement, SCS template attachment, and actor instance component duplication.
- `#3-live-widget-duplicate` `IN-REVIEW` tester — Partial live confirmation: `widget.duplicate` successfully cloned `HorizontalBox-Editor` to `HorizontalBox-Montage` in `/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu`, preserved nested `W_HotkeyInfo-Image_C` rows and slot data, returned a complete source-to-target mapping, and enabled the montage hotkey implementation. Actor/SCS component duplication was not tested in this session.
- `#4-verified-widget-scs-actor` `DONE` tester — Verified: `widget.duplicate` cloned `PanelA` to `PanelA_Copy` in `/Game/App/UI/Test/W_McpReviewTemp_20260429` with subtree mapping for `AlphaText`, `BetaText`, and `NestedHotkeyRow`; `blueprint.scs.duplicate_component` cloned `McpSourceScene` to `McpDupScene` in `/Game/App/UI/Test/BP_McpReviewTemp_20260429` and `blueprint.scs.get` showed both components; `actor.duplicate_component` cloned `McpSourceScene` to `McpInstanceDupScene` on `McpReviewTempActor_20260429` and `actor.get_components` showed the duplicated instance component.
