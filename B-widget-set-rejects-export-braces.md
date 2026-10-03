---
id: B-widget-set-rejects-export-braces
title: "`widget.set` rejects the `{…}` struct text that `widget.export_xml` emits (Brush, ColorAndOpacity, Font), though `widget.import_xml` now accepts it"
status: DONE
severity: Medium
category: bug
tags: [widget, widget-set, structs, export-text, braces, import-export-asymmetry]
encounters: 1
lastSeen: 2026-09-23T19:00:00Z
---

# The same brace string works in import_xml and fails in widget.set

Values copied from `widget.export_xml` (brace form) and passed to `widget.set` as strings:

```
widget.set {widgetName: "img_HeaderBG", properties: {Brush: "{TintColor={SpecifiedColor={R=1.0,...}},DrawAs=Image,...}"}}
 -> [INVALID_PROPERTY] Failed to set 'Brush': Unsupported string value for struct property 'Brush'
    (ImportText fallback: ImportText failed for property 'Brush' with value '{TintColor=...')
widget.set {widgetName: "tb_ProfileSub", properties: {ColorAndOpacity: "{SpecifiedColor={R=0.29,...},ColorUseRule=UseColor_Specified}"}}
 -> [INVALID_PROPERTY] Failed to set 'ColorAndOpacity': ... ImportText failed
```

The identical strings inside a `widget.import_xml` attribute were applied fine (the brace normalization
from `B-widget-xml-import-rejects-export-braces` #2 lives in `CoerceStringToJsonValueByProperty`'s struct
branch). `widget.set`'s string path goes straight to the ImportText fallback and never gets that rewrite.
Replacing every `{`/`}` with `(`/`)` makes both calls succeed.

**Workaround:** convert braces to parentheses before `widget.set`, or send nested JSON objects.

**Fix (proposed):** route `widget.set` struct strings through the same brace-to-paren normalization (one
shared helper for every ImportText fallback: `widget.set`, `property.set`, `blueprint.set_default`).

## History
- `#1-widget-set-brace-refused` `OPEN` reporter - Filed from a UMG pass on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Brace-form `Brush` / `ColorAndOpacity` / `Font` from `export_xml` refused by `widget.set`, accepted by `import_xml`; paren conversion fixed it. Related: `B-widget-xml-import-rejects-export-braces` (IN-REVIEW, import side only).
- `#2-brace-normalization-moved-to-shared-apply` `IN-REVIEW` developer — Confirmed at `10212ee4`: the brace→paren rewrite lived only in `CoerceStringToJsonValueByProperty` (import_xml / BPIR / blueprint default paths); `widget.set`, `property.set` pass the string straight to `ApplyJsonValueToProperty`, whose struct string branch handed `{...}` to `ImportText_Direct`. Moved the rewrite (one helper, `PropertyImportBraceHelpers::BraceHybridToExportText`) into that struct string branch's ImportText fallback, after the JSON parse fails; the coercion now leaves the string as-is, so every caller shares one path. Valid JSON and paren literals are unchanged. File: `Source/PinWright/Private/Utils/PropertyImport.cpp`. Test: `PinWright.widget.set.AcceptsExportedStructBraces` (`Source/PinWright/Private/Tests/WidgetXml/TestWidgetXmlExportReimport.cpp`) exports Image `Brush`/`ColorAndOpacity` and TextBlock `ColorAndOpacity`/`Font` (size 31), asserts each is brace form, `widget.set`s each unedited on a fresh WBP and compares with the source widgets; fails on revert. `PinWright.widget.import_xml.StructBraceRoundTrip` still covers the import path through the moved code. Docs: `docs/wiki-src/widget.md` (widget.set form 4), CHANGELOG.
- `#3-review-fixes` `IN-REVIEW` developer — Re-review: the shared test file `Source/PinWright/Private/Tests/WidgetXml/TestWidgetXmlExportReimport.cpp` dropped the 5.8-deprecated `GetObjectsWithOuter` bool overload (uses `MCP_FOREACH_EXCLUDE_NESTED_OBJECTS`); no behaviour change for this ticket. Test: `PinWright.widget.set.AcceptsExportedStructBraces`.
- `#4-verified-linux` `DONE` tester — Passed non-skipped in run3/full: `PinWright.widget.set.AcceptsExportedStructBraces` (exports Image `Brush`/`ColorAndOpacity` and TextBlock `ColorAndOpacity`/`Font`, asserts brace form, `widget.set`s each unedited and compares with the source), plus `PinWright.widget.import_xml.StructBraceRoundTrip` for the import path through the moved helper. Acceptance met: the brace-to-paren rewrite (`BraceHybridToExportText`) now sits in the shared `ApplyJsonValueToProperty` struct ImportText fallback, so `widget.set` and `property.set` accept the brace text `export_xml` emits, as `import_xml` does.
