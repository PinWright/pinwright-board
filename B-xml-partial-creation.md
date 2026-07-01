---
id: B-xml-partial-creation
title: "`widget_import_xml` partial creation on error leaves orphaned widgets"
status: DONE
severity: High
category: bug
tags: []
---

# `widget_import_xml` partial creation on error leaves orphaned widgets

When `widget_import_xml` fails mid-import (e.g., invalid slot property like `Padding` on `CanvasPanelSlot`), widgets created before the error persist in the tree. The error is returned but the partial state is committed. Subsequent retry fails with "Duplicate widget name". Must manually `widget_remove_widget` before retrying.

Combined with B-no-undo-redo (no undo/redo), there's no way to roll back the partial creation.

**Fix:** Import should be atomic — either all widgets succeed or none are created. Use a transaction or two-pass approach: validate first, then create.

## History
- `#1-orphaned-widget-on-error` `OPEN` reporter — W_Minimap was created in CanvasPanel_0 but slot Padding property failed. Widget persisted, retry errored with "Duplicate widget name". Had to remove_widget + retry.
- `#2-transaction-already-in-code` `IN-REVIEW` developer — Verified in code: `FScopedTransaction` wraps the entire import (line 634), and `GEditor->UndoTransaction()` is called on both construction failure (line 666) and binding failure (line 683). Validation pass (ValidateXmlNode) catches class/name/panel errors pre-construction. Property errors during ProcessXmlNode trigger rollback via undo. No code change needed — fix was already implemented, board was stale.
- `#3-returned-rollback-not-working` `OPEN` tester — Returned: rollback not working. Test on W_McpTestTemp2: added `<TextBlock Name="FailChild" Slot.BogusSlotProp="crash"/>` under TestPanel. Got CONSTRUCTION_FAILED error ("Property 'BogusSlotProp' not found on CanvasPanelSlot") but FailChild persisted in widget tree. `mcp__editor_automation__.call path="widget.describe" args={...}` confirms FailChild present after error. FScopedTransaction + UndoTransaction rollback path is not executing or not effective.
- `#4-transaction-cancel-fix` `IN-REVIEW` developer — Root cause: GEditor->UndoTransaction() inside FScopedTransaction scope — destructor calls EndTransaction() which re-commits the undone changes. Replaced with Transaction.Cancel() which sets Index=-1 so destructor skips EndTransaction(). Both construction failure (line 666) and binding failure (line 683) paths fixed.
- `#5-returned-cancel-insufficient` `OPEN` tester — Returned: partial creation still not rolled back. Test: `mcp__editor_automation__.call path="widget.import_xml" args={...}` on W_McpVerifyTemp with `<CanvasPanel Name="FailPanel"><TextBlock Name="FailChild" Slot.BogusSlotProp="crash"/></CanvasPanel>`. Got CONSTRUCTION_FAILED error but `mcp__editor_automation__.call path="widget.describe" args={...}` shows FailPanel+FailChild persisted. Transaction.Cancel() discards the transaction record but doesn't reverse in-memory widget construction — ConstructWidget+AddChild create new UObjects that aren't captured by FScopedTransaction. Need manual cleanup: collect widgets during creation, remove them from tree on error.
- `#6-manual-widget-removal-rollback` `IN-REVIEW` developer — Replaced transaction-based rollback with manual widget removal. Before ProcessXmlNode: save parent child count (add mode) or current root (root mode). On error: identify the imported root widget via index/root comparison, call RemoveWidgetSubtree to clean up entire partial subtree, then Transaction.Cancel(). Same pattern for binding failure path.
- `#7-verified-no-orphaned-widgets` `DONE` tester — Verified: `mcp__editor_automation__.call path="widget.import_xml" args={...}` on W_McpVerifyTemp with `<CanvasPanel Name="FailPanel"><TextBlock Name="FailChild" Slot.BogusSlotProp="crash"/></CanvasPanel>` in add mode under RootCanvas. Got CONSTRUCTION_FAILED error. `mcp__editor_automation__.call path="widget.describe" args={...}` shows only RootCanvas — FailPanel and FailChild were cleaned up by manual rollback. No orphaned widgets.
