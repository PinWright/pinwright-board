---
id: F-widget-reparent-placement
title: "`widget.reparent_widget` needs ordered placement"
status: DONE
severity: Medium
category: feature
tags: [widget-tree, reparent, placement, ordering, ergonomics]
---

# `widget.reparent_widget` needs ordered placement

`widget.reparent_widget` can move an existing child widget to another parent panel, but it has no way to control sibling order. The current implementation removes the widget from its old parent and calls `AddChild` on the new parent, so the moved widget is appended.

That is not enough for ordered UMG layouts. Sibling order can control draw order, focus/navigation order, layout order in boxes, and overlay stacking. Agents need to be able to move a widget before, after, or at a specific index in the target parent without reconstructing the tree.

## Desired Behavior

Extend `widget.reparent_widget` with the same `placement` object already used by `widget.duplicate`:

```json
{
  "path": "widget.reparent_widget",
  "args": {
    "widgetPath": "/Game/App/UI/W_Example.W_Example",
    "slotName": "WidgetToMove",
    "newParent": "TargetPanel",
    "placement": { "before": "ExistingSibling" }
  }
}
```

Supported placement forms:

```json
{ "index": 2 }
{ "before": "SiblingName" }
{ "after": "SiblingName" }
```

If `placement` is omitted, keep the current append-to-end behavior for compatibility.

## Proposed Fix

Move the placement resolver currently local to `WidgetDuplicateHandler.cpp` into a shared widget-authoring helper, then use it from both `widget.duplicate` and `widget.reparent_widget`.

`widget.reparent_widget` should resolve the insert index before mutation, remove the widget from its old parent, and attach it to the new parent with `InsertChildAt` instead of `AddChild`. Invalid `before` or `after` siblings should return `INVALID_PLACEMENT`. The handler should also reject moving a widget under itself or under one of its descendants.

## Workaround

There is no clean single-call workaround. Callers can sometimes rebuild the target subtree through XML import or remove/re-add widgets manually, but that risks losing authored slot data, bindings, descendants, or exact existing layout.

## History

- `#1-initial-request` `OPEN` reporter — User needed reparenting with exact order. Existing `widget.reparent_widget` only accepts `widgetPath`, `slotName`, and `newParent`, while source inspection showed it appends via `AddChild`. Requested a plan and board issue for ordered placement support.
- `#2-implemented-placement` `IN-REVIEW` developer — Added optional `placement` support to `widget.reparent_widget`, shared the existing duplicate placement resolver, switched reparent attach to `InsertChildAt` with the old slot as template, handled same-parent before/after index shifts, added descendant-parent validation, and added direct handler tests for before/index placement, same-parent ordering, slot preservation, invalid placement, and descendant-parent rejection.
- `#3-verified-before-placement` `DONE` tester — Verified: `widget.reparent_widget` moved `GammaText` from `PanelB` to `PanelA` in `/Game/App/UI/Test/W_McpReviewTemp_20260429` with `placement:{"before":"BetaText"}` and returned `insertIndex:1`; `widget.describe` on `PanelA` then showed child order `AlphaText`, `GammaText`, `BetaText`, `NestedHotkeyRow`.
