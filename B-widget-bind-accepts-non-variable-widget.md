---
id: B-widget-bind-accepts-non-variable-widget
title: "widget.bind accepts a binding on a widget with bIsVariable=false, which passes IsBindingValid and resolves to nothing at runtime"
status: IN-REVIEW
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
- `#2-not-a-defect-compiler-promotes-bound-widget` `IN-REVIEW` developer -- Investigated against
  engine source (5.3-5.8) rather than patched: the premise is wrong. `InitializeBindingsStatic` does
  resolve `ObjectName` through the generated class's object-property map, but
  `FWidgetBlueprintCompilerContext` generates a hidden variable for any widget a binding names --
  `bShouldGenerateVariable = Widget->bIsVariable || Widget->IsA<UNamedSlot>() ||
  WidgetBP->Bindings.ContainsByPredicate(Binding.ObjectName == Widget->GetName())`
  (`WidgetBlueprintCompiler.cpp` `PopulateBlueprintGeneratedVariables` on 5.6-5.8,
  `CreateClassVariablesFromBlueprint` on 5.3-5.5, comment verbatim: "In the event there are bindings
  for a widget, but it is not marked as a variable, make it one, but hide it from the UI"). Writing
  the record is what creates the property the runtime lookup needs, so the proposed refusal would
  have broken `widget.import_xml`'s `IsVariable="false"` output for no gain. No behaviour change.
  Added an explanatory comment at the resolution gate in
  `Handlers/UI/WidgetBindHandler.cpp` and a regression test
  `PinWright.widget.bind.NonVariableWidgetResolves`
  (`Tests/Widget/TestWidgetBindResolution.cpp`) that binds a `bIsVariable=false` widget, compiles,
  instantiates and runs `UUserWidget::Initialize`, then asserts the child's `TextDelegate` is bound to
  the handler -- so the refusal cannot be re-added silently. The ticket's suggested reuse is also
  unsound: see `B-widget-variable-guid-map-is-not-a-variable-set`.
