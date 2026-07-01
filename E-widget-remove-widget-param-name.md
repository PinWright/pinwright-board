---
id: E-widget-remove-widget-param-name
title: "widget.* namespace: 5 different param names for 'which widget'"
status: DONE
severity: Low
category: ergonomic
tags: []
---

# widget.* namespace: 5 different param names for the "which widget" selector

Across the `widget.*` namespace, handlers use five different names for what is
conceptually the same value — a `UWidget`'s instance/variable name inside a
widget blueprint. All resolve to `UWidgetTree::FindWidget(FName)` or an
equivalent name lookup. Result: callers must memorize a per-handler convention.

| Handler | Param (required/optional) | Resolves to |
|---|---|---|
| `widget.add` | `name` (req) | new child's `UWidget::GetFName()` |
| `widget.set` | `widgetName` (req) | `WidgetTree->FindWidget(FName)` |
| `widget.remove_widget` / `rename_widget` / `reparent_widget` | `slotName` (req) | `WidgetTree->FindWidget(FName(*SlotName))` — misleading: not a slot key |
| `widget.describe` | `widget_name` (opt, subtree root) | name lookup; BP path is `asset_path` + aliases `widgetPath`/`path`/`assetPath` |
| `widget.export_xml` | `widget_name` (opt, subtree root) | name lookup; BP path is `widgetPath` + aliases `widget_path`/`asset_path`/`path`/`assetPath` |
| `widget.import_xml` | `targetName` (opt, replace target or add-parent) | name lookup |

**Repro (remove_widget):** `name:"ReasonText"` →
`MISSING_REQUIRED_PARAM: Missing required parameter 'slotName'`. Retry with
`slotName:"ReasonText"` succeeds.

**Repro (export_xml):** `targetName:"ReasonText"` (mirroring `import_xml`) →
`UNKNOWN_PARAMS: Unknown parameter(s) for 'widget.export_xml': [targetName].`
`Valid parameters: [widgetPath, widget_path, asset_path, path, assetPath,`
`widget_name, include_defaults, resolve_geometry, resolveGeometry,`
`capture_source, verbose, include_geometry, omit_slot_chain, compact]`.
Canonical is `widget_name`.

**Proposal:** Standardize on `widgetName` as the canonical name everywhere in
the `widget.*` namespace, and accept `name`/`slotName`/`widget_name`/
`targetName` as aliases on every handler that takes this selector. Distinct
from `E-class-name-format-inconsistency` (class-name format drift, DONE) and
`E-blueprint-param-name-path-vs-assetpath` (blueprint asset-path drift) — this
is identifier-param naming drift inside `widget.*`.

## History
- `#1-filed` `OPEN` reporter — Hit during a session deleting `ReasonText` from
  `W_PhotoPopup`. Tried `name:` first (mirroring `widget.add`), got
  `MISSING_REQUIRED_PARAM`, retried with `slotName:` and it worked. Handler
  body uses `WidgetTree->FindWidget(FName(*SlotName))`, confirming the value
  is the widget's instance name, not a slot key.
- `#2-widened-to-export-import-describe` `OPEN` reporter — Hit the same drift on
  `widget.export_xml`: tried `targetName:` (mirroring `widget.import_xml`), got
  `UNKNOWN_PARAMS`; canonical is `widget_name`. Audited all widget-namespace
  handlers that take a widget-instance selector: 5 distinct names across 7+
  handlers (`name`, `widgetName`, `slotName`, `widget_name`, `targetName`).
  Verified against source: `WidgetAddHandler.cpp:22`, `WidgetSetHandler.cpp:112`,
  `WidgetHierarchyHandler.cpp:139,220,271` (remove/rename/reparent all use
  `slotName`), `WidgetDescribeHandler.cpp:176,267` (`widget_name`),
  `WidgetXmlExportHandler.cpp:37,126` (`widget_name`),
  `WidgetXmlImportHandler.cpp:545,570` (`targetName`). Title broadened from the
  remove-only framing to cover the whole namespace.
- `#3-standardized-widgetname-with-aliases` `IN-REVIEW` developer — Made `widgetName` the canonical selector across `widget.add`, `widget.set`, `widget.remove_widget`, `widget.rename_widget`, `widget.reparent_widget`, `widget.describe`, `widget.export_xml`, `widget.import_xml`. Each handler now declares the old names (`name`/`slotName`/`widget_name`/`targetName`/`target_name`) as `RPC_PARAM_OPT` aliases so the dispatcher's `UNKNOWN_PARAMS` validator accepts them, and reads the value via `GetStringFirstOf({widgetName, ...aliases})`. Required-spec was relaxed to optional with a manual `MISSING_PARAMETER` error when none of the aliases are supplied.
- `#4-verify-fix` `DONE` tester — Verified: schema-checked all 8 widget.* handlers (`add`/`set`/`remove_widget`/`rename_widget`/`reparent_widget`/`describe`/`export_xml`/`import_xml`) via `<method>?` discovery — each exposes `widgetName` plus `name`/`slotName`/`widget_name`/`targetName` aliases (import_xml also has `target_name`). Live-ran ticket repro `widget.export_xml{widgetPath:/App/App/UI/W_PhotoPopup, targetName:ReasonText}` on Unreal port 19880 — previously returned `UNKNOWN_PARAMS`, now returns valid XML (length=283).
