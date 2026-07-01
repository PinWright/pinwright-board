---
id: B-widget-export-xml-element-digit-prefix
title: "widget.export_xml emits XML element names starting with a digit (e.g. <1_MaterialLab_Transforms_C>), violates XML spec"
status: DONE
severity: Medium
category: bug
tags: [widget, export-xml, malformed-xml]
---

# `widget.export_xml` emits XML element names starting with a digit

`widget.export_xml` uses the UE child-widget class short name directly
as the XML element tag name. When a Blueprint class is named with a
leading digit (e.g. `1_MaterialLab_Transforms`,
`2_MaterialLab_Gradients`), the generated `tree.xml` contains element
tags like `<1_MaterialLab_Transforms_C>` — XML element names cannot
start with a digit per the XML 1.0 spec
(`NameStartChar` excludes `[0-9]`).

Currently 1 file in the cache hits this:
`Game\UIMaterialLab\Widgets\WBP_MaterialLab_Main\tree.xml` (9
instances). Any first-party content that adopts the same naming
convention will trip the same bug.

**Symptom:** Standard XML parsers reject the file with a syntax
error pointing at the digit-prefixed tag.

**Fix (proposed):** In the element-name emitter, detect names that
fail the XML `NameStartChar` rule and rewrite them. Mirror the
BPIR fix for names with spaces — add a safe prefix (e.g. `_1_MaterialLab...`)
or escape the leading digit with an entity reference, and record
the original name as an `original-name="..."` attribute for round-trip.

## History
- `#1-initial-audit` `OPEN` reporter — `WBP_MaterialLab_Main/tree.xml` contains 9 elements with names starting with a digit (`<1_MaterialLab_Transforms_C>` etc.); fails XML well-formedness check. Same root issue shape as `B-widget-export-xml-attribute-name-spaces` — element/attribute name sanitization.
- `#2-sanitize-element-tags` `IN-REVIEW` developer — Extended `WidgetXmlHelpers::SanitizeXmlName` (introduced by T1) to prepend `_` when the first char is a digit, `-`, or `.`. Sanitized element tag at WidgetXmlExporter.cpp:201 with companion `original-name` attribute for round-trip; WidgetXmlImportHandler.cpp prefers `original-name` over the literal tag for class resolution. Defensive sanitize in LiveUiSnapshotXmlWriter.cpp:328. Test in TestWidgetXmlHandlers.cpp.
- `#3-verify-fix` `DONE` tester — Verified: `widget.export_xml` on `/Game/UIMaterialLab/Widgets/WBP_MaterialLab_Main` emits sanitized tags like `<_1_MaterialLab_Transforms_C ... original-name="1_MaterialLab_Transforms_C">` for all 9 affected children. No element tag starts with a digit; XML well-formedness restored.
