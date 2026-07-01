---
id: F-widget-event-binding
title: "No way to create/restore widget event bindings via MCP"
status: DONE
severity: ""
category: feature
tags: []
---

# No way to create/restore widget event bindings via MCP

There is no MCP tool or BPIR syntax to create or restore the designer-level delegate binding between a widget (e.g., `Button.OnClicked`) and a Blueprint event node (`BndEvt__AcceptButton_OnClicked`). These bindings are normally created via the UE widget designer's "+" button in the Details panel. When `widget_import_xml` recreates widgets, existing BndEvt graph nodes lose their association with the new widget instances.

This breaks the full round-trip workflow: `widget_export_xml` → modify → `widget_import_xml` → broken event bindings → must fix manually in editor.

**Impact:** Any widget with event bindings (buttons, checkboxes, sliders) requires manual editor work after MCP-driven widget modifications. Blocks fully automated widget creation.

**Proposal:** Two complementary fixes:
1. **BPIR syntax:** Support `entry widget_event AcceptButton.OnClicked() { ... }` for creating BndEvt bindings during compilation. The decompiler already emits `entry override BndEvt__AcceptButton_OnClicked()` — round-trip requires the compiler to accept this and create the binding.
2. **widget_import_xml preservation:** When replacing/recreating widgets that had existing BndEvt bindings, auto-restore the delegate association on the new widget instances.

## History
- `#1-broken-runtime-binding` `OPEN` reporter — W_PhotoPopup buttons had BndEvt graph nodes with correct wiring but no runtime delegate binding after previous MCP widget modifications. Required manual rebinding in editor. Full MCP round-trip broken for any widget with event bindings.
- `#2-fixed-binding-init` `IN-REVIEW` developer — Two fixes: (1) CreateComponentEventNode now properly initializes UK2Node_ComponentBoundEvent — resolves FObjectProperty for component, finds FMulticastDelegateProperty for delegate, calls InitializeComponentBoundEventParams which sets correct DelegateOwnerClass (widget's class, not BP's generated class), EventReference, CustomFunctionName (BndEvt__ pattern), bInternalEvent, bOverrideFunction. Fixes runtime binding via RegisterDynamicBinding(). (2) New widget.bind_event handler verifies BndEvt node exists and is properly initialized, reports binding status, compiles BP.
- `#3-verified-delegate-valid` `DONE` tester — Verified: compile_bpir with `entry widget_event TestButton.OnClicked()` on W_McpVerifyTemp compiled clean (2 nodes). widget.bind_event returned customFunctionName:"BndEvt__W_McpVerifyTemp_TestButton_K2Node_ComponentBoundEvent_0_OnButtonClickedEvent__DelegateSignature", isDelegateValid:true. CustomFunctionName correctly set (was empty before fix). Node properly initialized via InitializeComponentBoundEventParams.
