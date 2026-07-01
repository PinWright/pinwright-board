---
id: B-widget-set-slot-struct-fails
title: "`widget.set` cannot set struct-valued slot properties"
status: DONE
severity: High
category: bug
tags: []
---

# `widget.set` cannot set struct-valued slot properties

`widget.set` with `slot` parameter cannot set struct-valued slot properties such as `CanvasPanelSlot.LayoutData` (which groups `Offsets`, `Anchors`, `Alignment`). All three reasonable JSON serializations are rejected:

1. **String literal** (matching decompiler output):
   ```json
   "slot": {"LayoutData": "(Offsets=(Left=210.0,Top=-100.0,Right=185.0,Bottom=40.0),Anchors=(Minimum=(X=0,Y=1),Maximum=(X=0,Y=1)),Alignment=(X=0,Y=0))"}
   ```
   → `INVALID_PROPERTY: Failed to set slot 'LayoutData': Unsupported JSON type for struct property`

2. **Dot-notation keys:**
   ```json
   "slot": {"LayoutData.Offsets.Left": 210, "LayoutData.Offsets.Top": -100, ...}
   ```
   → `INVALID_PROPERTY: Slot property not found: LayoutData.Offsets.Left`

3. **Nested JSON object:**
   ```json
   "slot": {"LayoutData": {"Offsets": {"Left": 210, "Top": -100}, "Anchors": {...}}}
   ```
   → `INVALID_PROPERTY: Failed to set slot 'LayoutData': Unsupported JSON type for struct property`

Non-struct slot properties (single-value primitives) work fine via `widget.set`. Only nested struct properties fail.

**Workaround:** Use `widget.import_xml` with `mode: add` and `targetName: <parent>`, passing the full subtree XML including `Slot.LayoutData="..."`. The XML path accepts the struct literal correctly — the gap is specifically in `widget.set`'s JSON-to-struct path.

**Discoverability issue:** `widget.add` + `widget.set` is the documented primary path for creating+configuring widgets. Since `widget.set` can't finish the job on a `CanvasPanelSlot`, agents hit a dead end unless they discover `widget.import_xml` accepts slot struct literals.

**Fix:** Accept struct-valued JSON in `widget.set`'s slot parameter, either by (a) parsing literal string form (reuse the XML handler's parser), (b) recursively setting sub-fields when given a nested object, or (c) documenting the gap and pointing to `widget.import_xml` in the error message.

## History
- `#1-three-json-formats-rejected` `OPEN` reporter — Tried three JSON formats in sequence while positioning a new `SaveButtonContainer` SizeBox on `W_HUD_RaceTrackEnd`'s CanvasPanel. All three failed. Deleted the partially-configured widget and switched to `widget.import_xml add` with the full XML subtree, which succeeded.
- `#2-struct-property-fallbacks-added` `IN-REVIEW` developer — Extended `ApplyJsonValueToProperty` (`Utils/PropertyUtils.cpp` lines 807-885) `FStructProperty` branch with two fallbacks. (1) For `EJson::Object` values, recursively applies each sub-field by walking the struct's `PropertyLink` chain with case-insensitive name match — handles nested JSON shape `{"LayoutData": {"Offsets": {"Left": 10, ...}}}`. (2) For `EJson::String` values, after the existing `JsonObjectToUStruct` attempt, falls through to `ImportTextToProperty` — handles ExportText literals like `"(Offsets=(Left=10,...))"` (same path widget_import_xml uses). Non-struct paths, Vector/Rotator array special case, and existing error semantics are unchanged.
- `#3-refactor-find-property-native` `IN-REVIEW` developer — Refactor pass: replaced inline `PropertyLink` walk with native `UStruct::FindPropertyByName(FName)` (FName equality is already case-insensitive). Consolidated 3 duplicate case-insensitive property helpers into single canonical `FindPropertyCI` in `Utils/PropertyUtils.h`; 5 call sites route through it. Added automation tests `FPropertyUtilsStructNestedJsonObjectTest` and `FPropertyUtilsStructExportTextLiteralTest`.
- `#4-verified-both-struct-shapes` `DONE` tester — Verified via MCP on `TestBox` SizeBox in a CanvasPanel: `widget.set slot:{"LayoutData":"(Offsets=(Left=10.0,Top=20.0,Right=100.0,Bottom=50.0),Anchors=(...),Alignment=(X=0,Y=0))"}` returned `propertiesSet:1` (ExportText literal path). `widget.set slot:{"LayoutData":{"Offsets":{"Left":55,"Top":66,"Right":77,"Bottom":88},"Anchors":{...},"Alignment":{"X":0.5,"Y":0.5}}}` returned `propertiesSet:1` (nested-object recursion path). Both shapes now accepted.
