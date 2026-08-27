---
id: B-widget-variable-guid-map-is-not-a-variable-set
title: "WidgetVariableNameToGuidMap holds every source widget, so widget.describe reports isVariable=true for all of them on 5.6+"
status: OPEN
severity: High
category: bug
tags: [umg, widget-describe, bIsVariable, WidgetVariableNameToGuidMap, import-xml, silent-wrong-data]
encounters: 1
lastSeen: 2026-08-28
---

# The map is a guid registry for every widget, not the set of blueprint variables

`WidgetDescribeHandler.cpp:447-453` treats presence in `UWidgetBlueprint::WidgetVariableNameToGuidMap`
as the 5.6+ equivalent of `UWidget::bIsVariable`:

```cpp
#if UE_VERSION_OLDER_THAN(5, 6, 0)
    const bool bWidgetIsVariable = Widget->bIsVariable;
#else
    const bool bWidgetIsVariable = WidgetBP->WidgetVariableNameToGuidMap.Contains(Widget->GetFName());
#endif
```

The map is not that set. `FWidgetBlueprintCompilerContext::ValidateAndFixUpVariableGuids`
(`WidgetBlueprintCompiler.cpp`) populates it from `WidgetBP->ForEachSourceWidget(...)` with no
`bIsVariable` test at all, and when the map is already non-empty it re-adds any missing widget under
`ensureAlwaysMsgf(..., TEXT("Widget [%s] was added but did not get a GUID"))`. `ForEachSourceWidget`
is `ForEachObjectWithOuter(WidgetTree, ...)` (`BaseWidgetBlueprint.cpp`), i.e. every widget in the
tree. The variable test the compiler itself uses is still the member: `bShouldGenerateVariable =
Widget->bIsVariable || Widget->IsA<UNamedSlot>() || <named by a binding>`, and only
`if (Widget->bIsVariable)` adds `CPF_BlueprintVisible`.

Consequences, in order of reach:

1. **`widget.describe` publishes `isVariable: true` for every widget after any compile**, including
   ones authored `IsVariable="false"`. Silent wrong data on a normal path — a caller deciding whether
   a `$Name` reference will resolve in BPIR gets a yes for a widget the compiler will never expose.
2. **`widget.import_xml`'s `RemoveWidgetVariableGuid` for non-variable nodes is undone and noisy.**
   `WidgetXmlImportHandler.cpp:784` removes the guid, then line ~791 calls
   `MarkBlueprintAsStructurallyModified`; the resulting compile re-adds the entry through the
   `ensureAlways` above. The removal never sticks, and it trips an engine ensure on the way.

`UWidget::bIsVariable` is a plain `UPROPERTY()` on `Components/Widget.h` in 5.6, 5.7 and 5.8 — not
deprecated, not editor-only — so the version gate has nothing to work around. The fix is to delete the
5.6+ branch and read `bIsVariable` unconditionally, and to drop the `RemoveWidgetVariableGuid` call
from the import path (the guid registry is the engine's, keyed on tree membership, not ours to prune).

**Not reproduced — source reading against `C:\UE_5.8` with 5.3/5.4/5.5/5.6/5.7 cross-checks.** Filed
by the agent working `B-widget-bind-accepts-non-variable-widget`, which was told to reuse this exact
version gate and found it unsound. That agent did not touch `WidgetDescribeHandler.cpp`.

## History
- `#1-map-is-per-widget-not-per-variable` `OPEN` reporter -- Filed while verifying
  `B-widget-bind-accepts-non-variable-widget`. Cites `ValidateAndFixUpVariableGuids`,
  `ForEachSourceWidgetImpl` and the compiler's own `bShouldGenerateVariable` /
  `CPF_BlueprintVisible` gates.
