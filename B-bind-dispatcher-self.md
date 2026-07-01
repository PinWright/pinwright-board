---
id: B-bind-dispatcher-self
title: "`bind_dispatcher` wires CreateDelegate.self to Target instead of self"
status: DONE
severity: Critical
category: bug
tags: []
---

# `bind_dispatcher` wires CreateDelegate.self to Target instead of self

`bind_dispatcher OnShowPhotoPopup(Target: %track, Delegate: OnPhotoPopupReceived)` wires `K2Node_CreateDelegate.self` to `%track` (delegate owner) instead of self (handler owner). Error: "Unable to find the selected function/event: 'OnPhotoPopupReceived'".

**Fix:** Wire `CreateDelegate.self` to `K2Node_Self` (or leave unconnected). Only `AddDelegate.Target` gets the external object.

## History
- `#1-initial-repro` `OPEN` reporter — Had to delete both AddDelegate+CreateDelegate nodes and rewire manually in editor.
- `#2-code-review-self-wiring` `IN-REVIEW` developer — Code review confirms BpirCompiler.cpp lines 3459-3472 wire CreateDelegate.Object to K2Node_Self, and lines 3377-3388 wire AddDelegate.Target to the external target. Test FCompilerIntegrationBindDispatcherExternalTargetUsesSelfTest validates both assertions. Fix was already implemented but board status not updated.
- `#3-verified-on-mcp-verify` `DONE` tester — Verified on `W_McpVerifyTemp` (temp widget with a `NamedBtn` CommonButton). Compiled BPIR `entry override Construct() { bind_dispatcher OnClicked(Target: $NamedBtn, Delegate: HandleClick); } entry function HandleClick() { call PrintString(InString: "Clicked"); }`. Inspected K2Node_CreateDelegate pins: `self` pin linked to `K2Node_Self_0:self` (confirmed via get_node_details — nodeType K2Node_Self). Inspected K2Node_AddDelegate pins: `self` pin (Target) linked to `NamedBtn` variable getter. Wiring is correct.
