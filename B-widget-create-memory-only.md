---
id: B-widget-create-memory-only
title: "`widget.create_widget_blueprint` reports a created asset after only marking/registering the package, with no disk-persistence result"
status: OPEN
severity: High
category: bug
tags: [widget, create, persistence, disk, false-success]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Widget blueprint creation is visible only in the warm editor

## What happens

After constructing the blueprint and root canvas,
`widget.create_widget_blueprint` marks/registers the package and calls
`McpSafeAssetSave` (`WidgetCreateHandler.cpp:149-172`). That shared helper is the
project's deliberate mark-dirty compatibility seam; it does not serialize the
package. The handler then returns `success:true`, says the widget blueprint was
created, and uses `AddAssetVerification` (`:174-189`), which verifies the resident
object/registry identity rather than a `.uasset` on disk.

The response has no `saveRequested`, `saved`, `pendingFlush`, `saveState`, package
filename, size or freshness field. A cold reload or editor exit without a later
explicit save can therefore lose the created asset despite the green creation
receipt.

## Why it matters

Creation is the first step of longer UI-authoring flows. Later successful edits can
accumulate on an asset that never became durable. Severity is High because the
empty initial blueprint is reconstructable, but the response silently hides the
persistence requirement.

## What should happen

Either use the Blueprint-safe durable-save path and return its complete measured
save report, or explicitly report the asset as memory-only/dirty with the required
follow-up action. Do not present registry re-resolution as disk proof. Add a cold
reload or package-file presence check to the acceptance test.

## Workaround

Call the explicit asset-save workflow after creation and verify the `.uasset`
exists on disk before performing long follow-up authoring or closing the editor.

## Related

- `B-widget-create-parent-fallback` — independent invalid-parent false success.
- `B-sequencer-create-save-no-disk-write` — established the same
  `McpSafeAssetSave`/registry-verification distinction for a creator.

## History
- `#1-source-scan-memory-only-create` `OPEN` reporter — Source-only scan followed widget creation through `McpSafeAssetSave`, `AddAssetVerification`, and the success response and found no package serialization or durability state. No build, test, editor, MCP call, or plugin edit was performed.
