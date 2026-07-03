---
id: E-remove-widget-required-bindwidget-no-warning
title: "widget.remove_widget silently removes a required BindWidget target, breaking the next compile with no upfront warning"
status: OPEN
severity: Medium
category: ergonomic
tags: [widget-remove-widget, bindwidget, compile, silent-success]
---

# widget.remove_widget silently removes a required BindWidget target, breaking the next compile with no upfront warning

`widget.remove_widget` deleted child `Settings_Panel`, which satisfies a REQUIRED `meta=(BindWidget)` property `UGameSettingPanel Settings_Panel` declared on the widget's C++ parent (`UGameSettingScreen`, a base of `ULyraSettingScreen`). The remove call returned success (`{"cascadedBoundEventsRemoved":0}`) with no indication the removed widget was a required bind target. The breakage only surfaced on the NEXT `blueprint.compile`: `A required widget binding "Settings_Panel" of type Game Setting Panel was not found.` Diagnosing the after-the-fact failure cost a wasted edit cycle plus an editor restart.

The handler (`Source/PinWright/Private/Handlers/UI/WidgetHierarchyHandler.cpp:144-226`) finds the widget by name (line 172) and removes it (`WidgetTree->RemoveWidget`, line 188) without ever checking whether its variable name matches a required `BindWidget` / optional `BindWidgetOptional` property on the generated-class parent chain. No helper in the plugin enumerates required BindWidgets — the parent-class metadata is available at remove time and could be inspected. The remove succeeds correctly; the gap is that a foreseeable destructive consequence is not surfaced until a separate compile fails.

**Workaround:** Before removing, reparent the widget off the required bind (rename the variable so it no longer matches the bind name), or keep a hidden placeholder widget under the bound name so the binding stays satisfied.
**Fix:** In the remove handler, walk the generated-class parent chain for properties with `BindWidget` (required) / `BindWidgetOptional`/`OptionalWidget` (optional) metadata; if the target's name matches a required bind, surface a warning field in the result (and the consequence) at remove time, or refuse unless an explicit override flag is passed.

## History
- `#1-initial-repro` `OPEN` reporter — Removed `Settings_Panel` from `/App/App/UI/LobbyAndMenu/W_DroneSelect_EditDrone` via `widget.remove_widget` → SUCCESS `{"cascadedBoundEventsRemoved":0}`; next `blueprint.compile` FAILED with `A required widget binding "Settings_Panel" of type  Game Setting Panel  was not found.` Verified against `WidgetHierarchyHandler.cpp:144-226`: removal at line 188 does no required-BindWidget check, and no helper enumerates required binds. Not a duplicate of `E-widget-remove-widget-cascade-bound-events` (bound-event cascade, DONE) or `E-widget-remove-widget-param-name` (selector naming, DONE).
