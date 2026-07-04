---
id: B-widget-xml-import-rejects-export-braces
title: "widget.import_xml rejects the {…} struct format that widget.export_xml emits"
status: IN-REVIEW
severity: Medium
category: bug
tags: [widget-xml, roundtrip, structs, import-export-asymmetry]
---

# `widget.import_xml` rejects the `{…}` struct format that `widget.export_xml` emits

`widget.export_xml` serializes struct-valued widget properties with **curly braces and `=` separators**, e.g. `BrushColor="{R=0.03,G=0.05,B=0.09,A=1.0}"` and nested `Slot.LayoutData="{Offsets={Left=0.0,...},Anchors={...}}"`. This form comes from `JsonValueToAttrString` for `EJson::Object` (`Handlers/UI/WidgetXmlUtils.h:149-159`), called from `WidgetXmlExporter.cpp:296`.

`widget.import_xml` accepts none of it. In `ApplyAttributeToObject` (`WidgetXmlImportHandler.cpp:251-275`): `CoerceStringToJsonValueByProperty` tries to parse the `{…}` as JSON (`Utils/PropertyImport.cpp:173-182`) but the brace form uses `=` and unquoted keys, so it is not valid JSON — it falls back to a raw string; `ApplyJsonValueToProperty`'s struct branch then re-parses as JSON (fails) and calls `ImportTextToProperty` (`PropertyImport.cpp:1072`), whose `ImportText_Direct` requires ExportText **parentheses** `(R=…)` and rejects braces. There is no `{}`→`()` normalization on any path, so an exported tree cannot be round-tripped back through import without hand-converting every brace. Observed: `export_xml` on a widget with a colored `Border` emits `BrushColor="{R=…}"`; feeding that back through `import_xml` returns `[CONSTRUCTION_FAILED] ... ImportText failed for property 'BrushColor' with value '{R=…}'`; replacing all `{}` with `()` succeeds.

**Workaround:** replace every `{`→`(` and `}`→`)` in the exported XML before importing (the `{key=value}` form maps exactly to a valid ExportText literal `(key=value)`).
**Fix:** make export/import agree on one syntax — either have the exporter emit the ExportText paren literal for structs, or have the importer normalize the `{key=value,…}` form (brace→paren, or parse it into a JSON object) before applying.

## History
- `#1-initial-repro` `OPEN` reporter — `export_xml` emits struct props as `{R=…}`/`{Offsets={…}}` (WidgetXmlUtils.h:149-159); `import_xml` rejects them because CoerceStringToJsonValueByProperty JSON-parse fails on `=`/unquoted keys and ImportText_Direct needs parens (PropertyImport.cpp:1072). Round-trip broken for every struct-valued property; braces→parens is the only workaround.
- `#2-import-side-normalization` `IN-REVIEW` developer — Verified the full trace in HEAD (export `{k=v}` at WidgetXmlUtils.h:149-160 → Exporter.cpp:296; coercion JSON-parse fail → FJsonValueString at PropertyImport.cpp:174-183; ImportText_Direct paren requirement at :1072; error string at :1076 matches the report). Fixed on the **import** side (lower blast-radius than changing `export_xml` output, which would ripple into asset-dump `tree.xml` caches / snapshot tests / the token-limit ticket): in `CoerceStringToJsonValueByProperty`'s `FStructProperty` branch, when a `{`-prefixed struct string fails JSON `Deserialize`, rewrite its braces to ExportText parens `(k=v)` (recurses to every nesting level) so `ImportText_Direct` applies it. Scoped to the struct branch, so string/name/text property values are never touched; the valid-JSON and paren-literal paths are untouched (no regression to the DONE B-widget-set-slot-struct-fails handling). Files: `Source/PinWright/Private/Utils/PropertyImport.cpp`. Test: `PinWright.widget.import_xml.StructBraceRoundTrip` (new file `Source/PinWright/Private/Tests/WidgetXml/TestWidgetXmlStructBraceImport.cpp`) drives the exact coerce→apply path on an in-code UImage fixture for a flat FLinearColor `{R=…}` and a nested FWidgetTransform `{Translation={X=…}…}`; fails-on-revert (apply returns false on the raw brace string).
