---
id: F-editor-utility-widget
title: "Spawn and run EditorUtilityWidget / EditorUtilityBlueprint"
status: DONE
severity: Low
category: feature
tags: [editor, utility-widget, blutility, ergonomic]
---

# Spawn and run EditorUtilityWidget / EditorUtilityBlueprint

The plugin has no RPCs for the Editor Utility (Blutility) pipeline.
`grep` over `docs/rpc-method-reference.generated.md` and
`Source/PinWright/Private/Handlers/` returns zero hits
for `EditorUtilityWidget`, `EditorUtilityBlueprint`, or
`UEditorUtilitySubsystem`. Today an agent can author a
`UEditorUtilityWidgetBlueprint` asset only by going through generic
asset/widget RPCs (no dedicated factory path) and has no way to register
the dockable tab or execute a `UEditorUtilityBlueprint::Run` action.
This blocks an end-to-end "build, deploy, and run a small editor tool"
loop entirely inside MCP, which pairs naturally with the existing
widget XML import / BPIR authoring surface.

Proposed handlers (all in `Handlers/Editor/` or a new
`Handlers/UtilityWidget/` subdirectory):

- `editor.create_utility_widget(path, parentClass?)` — creates a new
  `UEditorUtilityWidgetBlueprint` at `path`; `parentClass` defaults to
  `UEditorUtilityWidget`. Saves the package so subsequent widget XML
  imports / BPIR inserts can target it.
- `editor.spawn_utility_widget_tab(assetPath)` — loads the asset and
  calls `UEditorUtilitySubsystem::SpawnAndRegisterTab` (engine signature
  `SpawnAndRegisterTab(UEditorUtilityWidgetBlueprint*)`). Returns the
  registered tab id so callers can later close it.
- `editor.run_utility_blueprint(assetPath, params?)` — loads a
  `UEditorUtilityBlueprint` and invokes
  `UEditorUtilitySubsystem::TryRun` (the "Run Editor Utility Blueprint"
  context-menu action). `params` is optional and reserved for future
  argument forwarding once `TryRun` grows a parameter shape; today it
  is rejected if non-empty so the API can extend without breaking
  callers.

All three are mutating handlers and must follow the plugin's standard
game-thread / GC-deferred rules. Module dependency: add `Blutility`
(public module that hosts `UEditorUtilitySubsystem`,
`UEditorUtilityWidget`, `UEditorUtilityWidgetBlueprint`, and
`UEditorUtilityBlueprint`) to `PinWright.Build.cs`.
Test coverage mirrors `F-editor-status`: registration test plus a
valid-params smoke test that creates a transient utility widget BP and
spawns/closes the tab without ensure failures.

**Workaround:** None inside MCP today — the user has to open the editor
and right-click → Run Editor Utility Widget / Blueprint manually, or
script the same call via `python.execute` round-trips through
`unreal.EditorUtilitySubsystem`.

**Fix:** Add the three handlers above, reusing `AssetUtils` for path
resolution and the standard Blueprint creation helper for the new asset
factory path. No mutation of existing utility-widget assets is in scope
for this ticket — authoring still flows through widget XML / BPIR.

## History
- `#1-feature-request` `OPEN` reporter — No RPCs cover the Editor Utility (Blutility) pipeline: cannot create `UEditorUtilityWidgetBlueprint` assets, cannot call `UEditorUtilitySubsystem::SpawnAndRegisterTab`, cannot call `UEditorUtilitySubsystem::TryRun`. Confirmed by grepping `EditorUtilityWidget`, `EditorUtilityBlueprint`, `UEditorUtilitySubsystem`, `SpawnAndRegisterTab` in `docs/rpc-method-reference.generated.md` and `Source/.../Private/Handlers/` (zero hits). Severity Low — `python.execute` workaround exists.
- `#2-implemented-utility-widget-rpcs` `IN-REVIEW` developer — Added `editor.create_utility_widget` (FKismetEditorUtilities::CreateBlueprint with parent UEditorUtilityWidget::StaticClass and BP class UEditorUtilityWidgetBlueprint::StaticClass), `editor.spawn_utility_widget_tab` (UEditorUtilitySubsystem::SpawnAndRegisterTabAndGetID returning the FName tab id), and `editor.run_utility_blueprint` (UEditorUtilitySubsystem::TryRun, rejects non-empty params payload with INVALID_PARAMS for forward-compat). New file Handlers/Editor/UtilityWidgetHandler.cpp. Build.cs adds Blutility to PrivateDependencyModuleNames. Tests appended to Tests/EditorOps/TestEditorHandlers.cpp covering registration + asset-creation round-trip; counterfactual: replacing the create handler body with a no-op makes DoesAssetExist return false.
- `#3-verify-fix` `DONE` tester — Verified live: `editor.create_utility_widget` {name:"W_McpVerifyTemp_FeditorUtilityWidget", folder:"/Game/App/UI/Test"} returned success=true with assetClass=EditorUtilityWidgetBlueprint and existsAfter=true. `editor.spawn_utility_widget_tab` on that asset returned a tabId. `editor.run_utility_blueprint` with non-empty params={foo:"bar"} was rejected with INVALID_PARAMS ("'params' must be empty/omitted; argument forwarding to TryRun is not yet implemented."), matching the ticket contract. Temp asset deleted via `asset.delete`.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. One file basename renamed by the same plugin commit is repointed with it (`EditorAutomationRpcGateway.Build.cs` → `PinWright.Build.cs`, `EditorAutomationRpcGateway_SCSHandlers` / `_BlueprintHandlers_List` → `PinWright_*`), verified present at HEAD. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
