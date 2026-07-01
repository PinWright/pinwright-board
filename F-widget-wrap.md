---
id: F-widget-wrap
title: "`widget.wrap` — wrap an existing widget with a new parent without affecting its subtree"
status: DONE
severity: Medium
category: feature
tags: [widget-tree, widget-add, widget-reparent, ergonomics, wrap]
---

# `widget.wrap` — wrap an existing widget with a new parent without affecting its subtree

A common UMG authoring edit is: "take widget X and put it inside a new Overlay/Border/SizeBox Y, leaving X's children untouched." There's no single MCP call for this today.

## Current workaround

Compose via three calls:

1. `widget.add` — create the new wrapper `Y` under `X`'s current parent panel.
2. `widget.reparent_widget` — move `X` to live under `Y`.
3. `widget.set` — restore / adjust slot layout on `Y` (the wrapper inherits a fresh default slot on its new parent; `X`'s former slot is lost unless the caller remembers to re-apply it on `Y`).

Works for any non-root widget, but:
- Three round-trips instead of one.
- Caller is responsible for carrying `X`'s original slot layout to `Y`. Easy to forget, resulting in a wrapper at (0,0)/(100,30) default on a CanvasPanel.
- Doesn't work when `X` is the root widget (see `F-widget-set-root`).

## Proposal

Add a single handler:

```
widget.wrap
  widgetPath: string
  targetName: string     # the widget being wrapped
  wrapperType: string    # e.g. "Overlay", "Border", "SizeBox"
  wrapperName?: string   # optional; auto-generated if omitted
  wrapperProperties?: object   # initial properties for the wrapper
  wrapperSlot?: object         # slot properties on the new wrapper's parent;
                               # defaults to copying target's old slot so
                               # layout is preserved
```

Semantics:
- Insert `wrapperType` between `targetName` and its current parent.
- Copy `targetName`'s existing slot layout onto the wrapper so layout is preserved by default.
- Target's slot in its new parent (the wrapper) defaults to a sensible fill for the wrapper type (e.g. `HAlign_Fill`/`VAlign_Fill` for Overlay/Border).

## Why it matters

This is the natural shape of a common refactor: "this button needs a background now" (wrap in Border), "this row needs a fixed width" (wrap in SizeBox), "this content needs to center inside its parent" (wrap in Overlay). Today an agent has to compose it and get slot preservation right; one API makes it reliable.

## History
- `#1-reported` `OPEN` reporter — Hit while restructuring the root of a widget blueprint to replace `CanvasPanel` with `Overlay`. Needed to wrap the existing `HorizontalBox` root child with an `Overlay`, but the wrapper's slot was defaulted on creation and had to be re-set by hand from the original slot. Works but fragile; proposing `widget.wrap` as the ergonomic primitive.
- `#2-implemented-widget-wrap` `IN-REVIEW` developer — New handler `widget.wrap` added in `WidgetWrapHandler.cpp`. Uses `UPanelWidget::ReplaceChild` to preserve the original slot on the grandparent (anchors/offsets/alignment copied for free); adds target to wrapper with a fresh slot of wrapper's slot class, applies fill-defaults for Overlay/Border/SizeBox. Root-case explicitly rejected with INVALID_OPERATION (caller should use `widget.replace_class`). Pinned by `FWidgetWrapPreservesGrandparentSlotTest` in `TestWidgetWrapHandler.cpp`.
- `#3-verified-wraps-with-grandparent` `DONE` tester — Verified live on `/Game/App/UI/Test/W_McpVerifyTemp` (CanvasPanel root → TestText TextBlock). `mcp__editor_automation__.call path="widget.wrap" args={"targetName":"TestText","wrapperType":"Border","wrapperName":"TextBorder"}` returned `success: true, wrapperClass: "Border", grandparentName: "RootCanvas", targetName: "TestText", requiresCompile: true`. Post-call `mcp__editor_automation__.call path="widget.describe" args={...}` showed the tree as `CanvasPanel "RootCanvas" → Border "TextBorder" → TextBlock "TestText"` — Border now sits between RootCanvas and TestText with TestText as its child. Single-call wrap works as designed.
