---
id: B-utility-widget-memory-only
title: "`editor.create_utility_widget` reports a created asset after only marking/registering it, with no disk-persistence result"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, utility-widget, create, persistence, false-success]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Editor utility widget creation is visible only in the warm editor

## What happens

`editor.create_utility_widget` creates and registers the blueprint, calls the
mark-dirty-only `McpSafeAssetSave`, and then returns `success:true` with
`AddAssetVerification` (`UtilityWidgetHandler.cpp:101-139`). No package serializer
runs and the response has no durable save state, file presence, size or freshness.

The registry check proves that the same resident object can be found in the current
editor. It does not prove that a `.uasset` exists. Closing without a later explicit
save can therefore remove the asset despite the green creation receipt.

## Why it matters

Utility widgets are typically created so more logic can be authored and the asset
can be opened later. A caller can invest further work in a blueprint that was never
made durable. Severity is High because the initial empty asset is reconstructable,
but the response silently hides the persistence obligation.

## What should happen

Use the Blueprint-safe durable-save path and return a complete measured save report,
or explicitly label the result memory-only/dirty and name the required save action.
Acceptance coverage must verify package-file presence or cold reload, not only Asset
Registry visibility.

## Workaround

Run the explicit asset-save workflow immediately after creation and verify the
`.uasset` exists before authoring further logic or closing the editor.

## Related

- `B-widget-create-memory-only` — the runtime-widget creator has the same
  persistence gap.
- `F-editor-utility-widget` — feature ticket verifies warm creation only.

## Fix

Root cause: the handler's `McpSafeAssetSave` call only marked and registered the
new Blueprint, so its warm-editor verification was mistaken for disk
persistence. The handler now defaults `save` to true, uses
`SaveAssetToDiskReportingPresence`, and emits the shared
`saveRequested` / `saved` / `pendingFlush` report with `AssetSaveState` detail.
`save:false` deliberately retains the dirty in-memory package behavior. Parent
validation also now distinguishes unresolved from resolved-but-unusable classes
before package creation; the canonical `UEditorUtilityWidget` root remains
allowed even though the engine marks it abstract.

Files:

- `Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/UtilityWidgetHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/EditorOps/TestEditorHandlers.cpp`
- `Plugins/PinWright/Docs/wiki-src/editor.md`

Test IDs:

- `PinWright.editor.create_utility_widget.SaveWritesToDisk` — checks disk
  presence, registry visibility, and the save report.
- `PinWright.editor.create_utility_widget.InvalidParentRefused` — checks the
  typed incompatible-parent error and verifies no package, registry asset, or
  file.
- `PinWright.editor.create_utility_widget.SaveFalseIsMemoryOnly` — checks the
  not-requested state, dirty/pending-save response, and no disk file.

Deliberately unchanged: `editor.spawn_utility_widget_tab`,
`editor.run_utility_blueprint`, and existing assets/packages outside this create
flow. No build, test, editor, or MCP execution was performed; this ticket is
source-reviewed.

## History
- `#1-source-scan-utility-persistence` `OPEN` reporter — Source-only scan followed utility-widget creation through `McpSafeAssetSave`, `AddAssetVerification`, and success and found no package serialization or durability result. No build, test, editor, MCP call, or plugin edit was performed.
- `#2-durable-utility-widget` `IN-REVIEW` developer — Added durable default saving, typed parent validation, and handler-harness disk/registry/no-asset coverage. No build, test, editor, or MCP execution was performed.
