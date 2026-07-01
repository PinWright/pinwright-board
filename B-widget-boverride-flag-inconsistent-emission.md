---
id: B-widget-boverride-flag-inconsistent-emission
title: "tree.xml emits bOverride_* flags inconsistently across widgets"
status: DONE
severity: Low
category: bug
tags: [tree-xml, widget, schema]
---

# tree.xml emits bOverride_* flags inconsistently

Size override properties like `bOverride_WidthOverride="true"` appear on some `SizeBox` widgets in tree.xml but are completely omitted on others with similar setup. The dump can't distinguish "override disabled (false)" from "property not dumped".

## Samples

- Present: `App/App/UI/LobbyAndMenu/Popups/W_SaveTrack/tree.xml` — bOverride_WidthOverride emitted on most SizeBoxes
- Absent: `App/App/UI/LobbyAndMenu/Settings/W_HorizontalSelector/tree.xml` — bOverride_* flags missing

## Fix sketch

In the widget property collector (`WidgetXmlExporter.cpp::CollectOverriddenAttributes`), explicitly preserve UPROPERTY-marked `bOverride_*` boolean flags regardless of value (don't elide when false). Alternatively, document that absence = false at the consumer level.

## History
- `#1-boverride-inconsistent` `OPEN` reporter — schema ambiguity. Fix is either always-emit or document absence semantics.
- `2026-05-22` implementation — `WidgetXmlExporter.cpp::CollectOverriddenAttributes` now preserves reflected `bOverride_*` bools through sparse/default-elision export, including explicit `false`; regression coverage added under `Tests/WidgetXml`.
- `#2-verify-fix` `DONE` tester — Verified: re-dumped `/App/App/UI/LobbyAndMenu/Settings/W_BlackCheckbox` and `/App/App/UI/LobbyAndMenu/Popups/W_SaveTrack`; tree.xml now emits all 8 SizeBox `bOverride_*` flags including `false` values (e.g. `bOverride_HeightOverride="false"`, `bOverride_MinDesiredWidth="false"`, etc. alongside `bOverride_WidthOverride="true"`). W_HorizontalSelector has no SizeBox so n/a there.
