---
id: B-widget-rename-false-success
title: "`widget.rename_widget` ignores UObject rename failure and echoes the requested name as a successful result"
status: OPEN
severity: High
category: bug
tags: [widget, rename, validation, readback, false-success]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Widget rename reports the request instead of the achieved object name

## What happens

`widget.rename_widget` checks only that the old widget exists. It does not reject an
existing destination name or validate the new UObject name. It then calls
`TargetWidget->Rename(*NewName)` and discards the boolean result
(`WidgetHierarchyHandler.cpp:280-309`). Regardless of what the object is actually
named, the response sets `success:true` and copies the requested `newName`
(`:311-328`).

When the target name is occupied or invalid, Unreal can refuse or alter the rename.
The handler still removes/recreates the variable GUID using the actual object name,
marks the blueprint structurally modified, and tells the caller the requested name
landed.

## Why it matters

Later calls address a widget name that may not exist, while the blueprint has still
been dirtied and its GUID bookkeeping touched. Severity is High for a normal
authoring false success.

## What should happen

Validate the requested name and reject collisions before opening the transaction.
Check the `Rename` result, cancel/restore on failure, and return the actual
`TargetWidget->GetName()` only after a tree lookup proves the new identity resolves
and the old identity does not.

## Workaround

Use `widget.describe` to ensure the destination is unused, then call it again and
trust only the returned tree, not `rename_widget.newName`.

## Related

- `E-remove-widget-required-bindwidget-no-warning` — binding advisory, not rename
  outcome validation.

## History
- `#1-source-scan-rename-readback` `OPEN` reporter — Source-only scan confirmed the rename return value is discarded, destination validity/collision is not checked, and the response echoes the request rather than object state. No build, test, editor, MCP call, or plugin edit was performed.
