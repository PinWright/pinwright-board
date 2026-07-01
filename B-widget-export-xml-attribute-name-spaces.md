---
id: B-widget-export-xml-attribute-name-spaces
title: "widget.export_xml emits XML attribute names with spaces, producing files unparseable by any standard XML parser"
status: DONE
severity: High
category: bug
tags: [widget, export-xml, malformed-xml]
---

# `widget.export_xml` emits malformed XML for properties whose names contain spaces

`widget.export_xml` serializes UE widget property and variable names
verbatim as XML attribute names. When the property display name or
variable name contains a space — e.g. `StabilizationMode Text`,
`Axis Type`, `Gradient Label`, `Show Seconds`, `Font Size` — the
resulting XML attribute name violates the XML spec. Standard XML
parsers (`xml.etree.ElementTree`, `lxml`, browser DOM, etc.) reject
the file outright.

5 of 6 affected files in the cache fail to parse with
`xml.etree.ElementTree`. The 6th squeaks through only because a
parser quirk masks the violation.

**Repro:**
- `App\App\UI\LobbyAndMenu\Settings\W_ControllerAxesPanel\tree.xml`
  (first-party App widget) — `StabilizationMode Text` and `Axis Type`
  attributes, 8 occurrences each.
- `Game\UltraDynamicSky\Blueprints\Widgets\UDS_Onscreen_Controls\tree.xml`
  and 3 sibling UDS widgets.

**Symptom:** Any consumer trying to parse `tree.xml` for these widgets
gets an XML syntax error and cannot proceed.

**Fix:** Replace any character outside `[A-Za-z0-9_.-]` with `_` in
`WidgetXmlHelpers::SanitizeXmlName` (added to `WidgetXmlUtils.h`). All
attribute keys in `WidgetXmlExporter.cpp` (property names at lines 56-58,
binding keys at line 230) pass through this sanitizer before emission.
`WidgetXmlImportHandler.cpp` adds a defensive `_`↔space retry at both
`FindPropertyCI` call sites so legacy FNames that truly contained spaces
still round-trip correctly.

The `original-name="..."` companion-attribute proposal was rejected: it
provides the same well-formedness guarantee at the cost of significantly
larger output files, and no current consumer needs the original name for
anything other than property lookup (which the retry covers).

## History
- `#1-initial-audit` `OPEN` reporter — 6 cached `tree.xml` files contain attribute names with embedded spaces; 5 fail `xml.etree.ElementTree.parse()`. Affects first-party `W_ControllerAxesPanel` plus four UDS plugin widgets.
- `#2-sanitize-attr-names` `IN-REVIEW` developer — Added `WidgetXmlHelpers::SanitizeXmlName` in WidgetXmlUtils.h. Sanitized attribute keys at WidgetXmlExporter.cpp:56-58 and :230. Added defensive `_`↔space fallback in WidgetXmlImportHandler.cpp for round-trip. Tests in TestWidgetXmlHandlers.cpp. Rejected the original-name companion-attribute approach: same well-formedness gain, much larger files.
- `#3-verify-fix` `DONE` tester — Verified: ran `widget.export_xml` on `/App/App/UI/LobbyAndMenu/Settings/W_ControllerAxesPanel`. All previously-offending attributes now use underscores (`StabilizationMode_Text`, `Axis_Type`, 8 occurrences each); zero spaces in any attribute name across 88 widgets in returned XML.
