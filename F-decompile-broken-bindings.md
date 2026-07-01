---
id: F-decompile-broken-bindings
title: "Decompiler should warn on broken BndEvt widget delegate bindings"
status: DONE
severity: ""
category: feature
tags: []
---

# Decompiler should warn on broken BndEvt widget delegate bindings

`blueprint_decompile` emits `entry override BndEvt__AcceptButton_OnClicked()` for BndEvt event nodes, but cannot detect whether the corresponding widget's delegate is actually bound at the designer level. When a widget is recreated (e.g., via `widget_import_xml`), the BndEvt graph node survives but the widget-level delegate association breaks. The agent sees correct-looking BPIR but the button doesn't fire at runtime.

**Proposal:** During decompilation, for each `BndEvt__<WidgetName>_<EventName>` node, verify that the widget variable `<WidgetName>` exists AND has the `<EventName>` delegate bound. If not, emit a warning: `"BndEvt__AcceptButton_OnClicked: delegate binding not found on widget 'AcceptButton'"`. This catches broken bindings that are invisible to graph-level inspection.

**Impact:** Would have caught the W_PhotoPopup button binding bug (buttons didn't fire OnClicked despite correct graph wiring).

## History
- `#1-silent-broken-binding` `OPEN` reporter — W_PhotoPopup AcceptButton/RejectButton BndEvt nodes existed with correct graph wiring (verified via get_execution_flow, get_pin_details), but runtime button clicks did nothing. User had to diagnose by inspecting the editor manually.
- `#2-added-binding-validation` `IN-REVIEW` developer — After emitting entry signature for UK2Node_ComponentBoundEvent, validates widget/component existence and delegate validity via IsDelegateValid(). Adds warning to decompile result if broken. Checks WidgetTree for widget BPs, GeneratedClass properties for actor BPs.
- `#3-verified-binding-warning` `DONE` tester — Verified: created BndEvt node for TestButton.OnClicked on W_McpVerifyTemp, then removed TestButton widget. blueprint_decompile returned warning: "widget_event TestButton.OnClicked: widget variable 'TestButton' not found — the widget may have been removed or recreated without recompiling". Warning absent when widget exists.
