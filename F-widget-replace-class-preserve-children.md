---
id: F-widget-replace-class-preserve-children
title: "`widget.replace_class` — change a widget's class (including the root) while preserving its children"
status: DONE
severity: High
category: feature
tags: [widget-tree, widget-replace, widget-root, widget-import-xml, ergonomics, class-swap]
---

# `widget.replace_class` — change a widget's class (including the root) while preserving its children

A UMG refactor like "change this `CanvasPanel` to an `Overlay`" or "change this `Border` to a `SizeBox`" — including when the target is the widget blueprint's root — has no direct MCP path that keeps the children attached where they are.

The root case and the non-root case are the same operation conceptually: replace one panel widget with another while preserving the subtree. The only difference is whether the target has a parent slot to migrate. Both should be served by a single handler.

## Current state

- `widget.remove_widget` deletes the widget **and its subtree** — children go with it.
- `widget.import_xml` without `targetName` replaces the whole tree (so it *does* change the root class), but returns `requiresCompile: true` and `variablesCreated: N`, meaning every widget is a new instance with a new GUID. BP event-graph nodes that resolve by FProperty name on the generated class rebuild after recompile, but widget-GUID-bound references are stale. Destructive.
- `widget.import_xml` with `targetName` has the same drawback one level in: the named widget *and* its subtree are rewritten to new instances.
- `python.execute` **cannot** set `UWidgetTree::RootWidget` — the property is `BlueprintProtected`, unreal-python blocks reads and writes with `"Property 'RootWidget' for attribute 'RootWidget' on 'WidgetTree' is protected and cannot be read/set"`. `panel.add_child` / `panel.remove_child` work on non-root panels; the actual root assignment does not.
- Manual compose via `widget.add` + `widget.reparent_widget` can build the desired *structure*, but for the root case it produces deeper nesting (new wrapper under the unchanged old root) rather than an in-place root swap.

## Workaround (non-root)

Compose the swap:

1. `widget.add` — create the replacement widget `Y` as a sibling of the original `X` under `X`'s parent.
2. For each child of `X`: `widget.reparent_widget` into `Y`.
3. `widget.set` — copy properties from `X` to `Y` where they share names (background, padding, etc.).
4. `widget.remove_widget` — delete `X` (now empty).
5. `widget.set` — re-apply `X`'s former slot layout to `Y` on the parent.

Multiple round-trips, easy to mis-order, slot-preservation is manual. Does not apply to the root.

## Workaround (root)

Only `widget.import_xml` without `targetName`, which recreates every widget instance. Acceptable only when the blueprint has no BP event bindings to preserve.

## Proposal

```
widget.replace_class
  widgetPath: string
  targetName: string              # widget to replace; may be the root
  newType: string                 # target class, e.g. "Overlay"
  preserveProperties?: bool       # default true; copy matching properties
  preserveSlot?: bool             # default true; for non-root, preserve
                                  # target's slot layout on its parent
                                  # (no-op for root)
```

Semantics:
- Allocate a new widget of `newType` with the same name as `targetName` (so BP references resolving by name continue to work after recompile).
- Move every child of the target into the new widget, preserving each child's existing slot state where slot types match between old and new panels; otherwise fall back to a sensible default for the new panel type.
- Copy overridden properties from the target to the replacement for properties that exist on both classes.
- If `targetName` is the root: assign `WidgetTree.RootWidget = new` and dispose the old root.
- If `targetName` is not the root: preserve its parent-slot layout on the parent panel, then delete the old widget.
- Mark the BP dirty. Caller recompiles.

Implementation notes:
- The operation requires C++ access to `UWidgetTree::RootWidget` — python is blocked.
- Widget-GUID-based BP references will still break across the swap; name-based FProperty references survive.

## Why it matters

- Class swaps are frequent during iteration — a CanvasPanel used as a single-child root is overkill; a Border becomes a SizeBox when shape constraints change; an Overlay gains an explicit anchor and needs to be a CanvasPanel.
- Project conventions in many UE codebases prefer `Overlay` / `HorizontalBox` / `VerticalBox` as widget roots over `CanvasPanel` for performance (CanvasPanel's per-child positioning evaluation is non-trivial). Newly-authored widgets default to CanvasPanel in the UMG designer, so the "flatten to Overlay root" cleanup is a standard refactor.
- Today an agent either does the non-root compose (slow, error-prone) or falls back to `widget.import_xml` on the whole subtree (destructive), and for root swaps only the destructive path exists.

## History
- `#1-reported` `OPEN` reporter — Hit while trying to swap the root panel class on several widget blueprints from `CanvasPanel` to `Overlay`. Verified all current approaches and their failure modes: python blocked by protected `RootWidget` property; `import_xml` works but recreates every widget instance; compose via add+reparent leaves the old root in place for the root case. Filing unified tool request covering both root and non-root swaps (initially filed as two separate issues — merged because the root case is not a distinct operation, just one where there is no parent slot to preserve).
- `#2-implemented-replace-class` `IN-REVIEW` developer — New handler `widget.replace_class` in `WidgetReplaceClassHandler.cpp`. Supports non-root (via `UPanelWidget::ReplaceChild` preserving parent slot) and root (direct `WidgetTree->RootWidget` native write — Python-binding block doesn't apply to C++). Children migrated via `UPanelWidget::AddChild(Widget, SlotTemplate)` so matching-panel slot props carry across. Added shared `WidgetAuthoringHelpers::CopyMatchingProperties` utility. Pinned by `FWidgetReplaceClass_NonRoot_PreservesChildrenAndParentSlot` and `FWidgetReplaceClass_Root_ReassignsRootWidget` in `TestWidgetReplaceClassHandler.cpp`.
- `#3-verified-root-swap-keeps-children` `DONE` tester — Verified live on `/Game/App/UI/Test/W_McpVerifyTemp` (root CanvasPanel with 3 descendants: HudCheck, TextBorder→TestText). `mcp__editor_automation__.call path="widget.replace_class" args={"targetName":"RootCanvas","newType":"Overlay"}` returned `success: true, oldClass: "CanvasPanel", newClass: "Overlay", wasRoot: true, requiresCompile: true`. Post-call `mcp__editor_automation__.call path="widget.describe" args={...}` showed the tree as `Overlay "RootCanvas" → CheckBox "HudCheck" + Border "TextBorder" → TextBlock "TestText"` — root class swapped in place, both immediate children preserved, deep child preserved through the wrapper. The root case (which has no python-binding workaround) works.
