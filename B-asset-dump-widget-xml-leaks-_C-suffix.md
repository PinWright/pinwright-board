---
id: B-asset-dump-widget-xml-leaks-_C-suffix
title: "asset.dump tree.xml emits BlueprintGeneratedClass _C suffix as XML element tag name"
status: DONE
severity: Low
category: bug
tags: [asset-dump, widget, tree-xml]
---

# asset.dump tree.xml emits BlueprintGeneratedClass _C suffix as XML element tag name

BlueprintGeneratedClass internal `_C` suffix becomes the XML element tag name for nested user-widget instances. Inconsistent with native widget tags (no `_C`) and produces ugly read-back to consumers / tooling.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/UI/Foundation/Buttons/W_LyraButton/tree.xml`.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/UI/LobbyAndMenu/HUD/W_HUD_Race_Main/tree.xml`.
3. Observe: tags like `<InputActionWidget_C name="InputActionWidget" ...>` (literal `_C` suffix in element name).

**Fix:** Extended `StripClassPrefix` in `WidgetXmlUtils.h` to also `RemoveFromEnd(TEXT("_C"))` after the existing leading-`U` strip. Fixes the tag-name leak in `WidgetXmlExporter.cpp` (the only consumer of the shared helper). Import side untouched: `ResolveWidgetClassFromTag` (`WidgetXmlImportHandler.cpp:36-46`) already retries with `+"_C"` on lookup failure, so symmetry is preserved. Note: `WidgetDescribeHandler.cpp` and `LiveUiSnapshot.cpp` each carry their own anonymous-namespace `StripClassPrefix` and still leak `_C`; consolidating those callers onto the shared helper is a follow-up.

## History
- `#1-initial-repro` `OPEN` reporter — Generated user-widget classes appear in tree.xml as `<Foo_C ...>` because `_C` is preserved by `StripClassPrefix` (`WidgetXmlExporter.cpp:201`). Sample paths: `Game/UI/Foundation/Buttons/W_LyraButton/tree.xml`, `App/App/UI/LobbyAndMenu/HUD/W_HUD_Race_Main/tree.xml`.
- `#2-strip-_C-suffix-in-shared-helper` `IN-REVIEW` developer — Extended `StripClassPrefix` (`WidgetXmlUtils.h`) to drop trailing `_C` after the existing leading-`U` strip. The only consumer of the shared helper is `WidgetXmlExporter.cpp`. Import remains compatible — `ResolveWidgetClassFromTag` already retries `+"_C"`. Added regression test `FXmlExportUserWidgetTagDropsCSuffixTest` building a compiled transient user widget and asserting its XML tag has no `_C` suffix. `WidgetDescribeHandler.cpp` and `LiveUiSnapshot.cpp` keep their own private `StripClassPrefix` copies (anonymous namespaces) — separate follow-up to consolidate.
- `#3-verify-fix` `DONE` tester — Verified: re-ran `asset.dump` on both repro assets (`/Game/UI/Foundation/Buttons/W_LyraButton`, `/App/App/UI/LobbyAndMenu/HUD/W_HUD_Race_Main`). `grep -E "<[A-Za-z_]+_C[ />]"` returned no matches on either `tree.xml`; the `InputActionWidget` instance in `W_LyraButton/tree.xml:9` now emits `<InputActionWidget name="InputActionWidget" ...>` (no trailing `_C`). Remaining `_C` substrings are inside attribute values referencing class paths, which are legitimate.
