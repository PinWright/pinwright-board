---
id: E-xml-unnamed-widgets
title: "`widget.import_xml` should warn when widgets lack explicit names"
status: DONE
severity: ""
category: ergonomic
tags: []
---

# `widget.import_xml` should warn when widgets lack explicit names

Agents rarely set the `name` attribute on widget elements. When omitted, the handler silently auto-generates a name from the class (e.g. `Button`, `CanvasPanel_1`). This breaks: (1) `add`-mode re-imports — auto-names collide causing `Duplicate widget name` errors; (2) BPIR `$WidgetName` references and `IsVariable` promotion require stable names; (3) event bindings (`BndEvt__*`) break when auto-names shift on re-import; (4) `export_xml` → edit → `import_xml` round-trips silently lose widget identity.

**Fix:** Emit a `warnings` array entry (matching the existing warnings convention used by Blueprint/Actor/Animation handlers) plus a `UE_LOG(LogTemp, Warning, ...)` when any element is missing `name`. Import still succeeds.

## History
- `#1-unnamed-collisions-repro` `OPEN` reporter — Agents omit `name=` on most widget elements. Re-imports on `add` mode fail with "Duplicate widget name". BPIR `$Name` refs resolve incorrectly when auto-names shift between imports.
- `#2-added-autoname-warnings` `IN-REVIEW` developer — Added `TArray<FString>& OutAutoNamedWidgets` out-param to `ValidateXmlNode` (and its recursive child calls). When `name` is absent, the generated name is appended to this array. Call site (`widget.import_xml` handler, step 6) declares a local `AutoNamedWidgets` and passes it in. After successful construction (step 12), if non-empty: builds a single-string `warnings` array entry with the count, the auto-generated name list, and all four risk reasons; sets `ResultObj->SetArrayField("warnings", ...)` and emits `UE_LOG(LogTemp, Warning, ...)`. Field is absent when all widgets have explicit names. Implementing file: `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Handlers/UI/WidgetXmlImportHandler.cpp`.
- `#3-verified-warnings-array` `DONE` tester — Verified on `W_McpVerifyTemp`. Imported XML with 3 unnamed elements (VerticalBox + 2 TextBlocks) and 2 named (RootCanvas, NamedBtn). Response contained `warnings` array with a single entry: `"3 widget(s) imported without an explicit 'name' attribute and received auto-generated names: [VerticalBox, TextBlock, TextBlock_1]. ..."` including all four risk reasons. Count matched expected.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
