---
id: B-widget-set-rejects-export-braces
title: "`widget.set` rejects the `{…}` struct text that `widget.export_xml` emits (Brush, ColorAndOpacity, Font), though `widget.import_xml` now accepts it"
status: OPEN
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
