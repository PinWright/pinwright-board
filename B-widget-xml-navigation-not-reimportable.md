---
id: B-widget-xml-navigation-not-reimportable
title: "widget.export_xml emits Navigation with FDelegateProperty annotations that widget.import_xml cannot parse (CONSTRUCTION_FAILED: ImportText left trailing input)"
status: OPEN
severity: Medium
category: bug
tags: [widget, export-xml, import-xml, round-trip, navigation, widget-navigation, delegate-property]
encounters: 1
lastSeen: 2026-09-23T18:30:00Z
---

# export_xml Navigation does not round-trip through import_xml

`widget.export_xml {compact: true}` of `/App/App/UI/LobbyAndMenu/Popups/W_LoginWidget` emits on an EditableTextBox:
`Navigation="{Down={Rule=Explicit,WidgetToFocus=PassTextBox,Widget=,CustomDelegate={_kind=FDelegateProperty,type=FCustomWidgetNavigationDelegate,bindings=,bindingStatus=empty}},...,_kind=/Script/UMG.WidgetNavigation}"`.
Re-importing that element with `widget.import_xml` fails the whole import with `CONSTRUCTION_FAILED: Failed to set property 'Navigation' on widget 'FirstNameBox': ImportText left trailing input`. The `_kind` / `bindings` / `bindingStatus` annotations are not ImportText syntax.

**Workaround:** strip every `Navigation` attribute before import (loses explicit Tab/arrow navigation).

**Fix:** emit ImportText-compatible `Navigation` (or omit empty CustomDelegate annotations), or make import ignore the annotation keys.

**Source (8748c637):** the delegate annotation is `MakeSingleDelegateMarker` (`Utils/PropertyExport.cpp:231-250`, reached from `:934`), a JSON object with `_kind` / `type` / `bindings` / `bindingStatus` keys intended for readback, flattened into the XML attribute. The import-side brace-to-paren normalization from `B-widget-xml-import-rejects-export-braces` (IN-REVIEW) makes `{k=v}` ImportText-legal but cannot make these marker keys legal, so this is a separate round-trip gap.

## History
- `#1-navigation-breaks-import` `OPEN` reporter — UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`, building W_AppSchoolNameLogin from W_LoginWidget's exported tree.
