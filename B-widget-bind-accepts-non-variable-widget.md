---
id: B-widget-bind-accepts-non-variable-widget
title: "widget.bind accepts a binding on a widget with bIsVariable=false, which passes IsBindingValid and resolves to nothing at runtime"
status: OPEN
severity: High
category: bug
tags: [umg, widget-bind, bIsVariable, InitializeBindingsStatic, silent-false-success, import-xml]
encounters: 1
lastSeen: 2026-08-27
---

# One false-success vector left in the verb, reachable through widget.import_xml

`B-widget-bind-event-suffix-never-binds` is fixed: `propertyName` is now resolved through UMG's own two
lookups and a candidate record is run through `FDelegateEditorBinding::IsBindingValid` before anything
is written.

One gap survives that check. `IsBindingValid` only verifies the widget exists via
`WidgetTree->FindWidget`. A widget with `bIsVariable = false` passes that -- but
`UWidgetBlueprintGeneratedClass::InitializeBindingsStatic` looks the object up in the generated class's
**widget property map**, which a non-variable widget is not in. So the binding is written, validates,
and resolves to nothing at runtime.

Reachable in practice: `widget.import_xml` can produce non-variable widgets
(`WidgetXmlImportHandler.cpp:472`).

**Fix:** add the variable test to the resolution gate. The version-gated form already exists in the
tree -- `WidgetDescribeHandler.cpp:447-453` uses `WidgetVariableNameToGuidMap` on 5.6+ and
`bIsVariable` below -- so this is reuse rather than new engine work. The refusal should say the widget
must be a variable, since the remedy is not obvious.

## History
- `#1-remaining-vector-after-the-resolution-fix` `OPEN` reporter -- Recorded by the agent fixing
  `B-widget-bind-event-suffix-never-binds`, which left it out to keep that diff on the ticket's
  subject. Source-level claim against `InitializeBindingsStatic` and `IsBindingValid`.
