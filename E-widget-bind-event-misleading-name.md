---
id: E-widget-bind-event-misleading-name
title: "`widget.bind_event` name implies creation but only verifies"
status: DONE
severity: Low
category: ergonomic
tags: []
---

# `widget.bind_event` name implies creation but only verifies

`widget.bind_event` sounds like it *creates* a delegate binding. It actually requires a pre-existing `BndEvt__*` K2Node (created via `compile_bpir widget_event X.Y()`), and only verifies/reports its binding state (`isDelegateValid`, `customFunctionName`). Calling it without the node returns `BPIR_REQUIRED: No BndEvt node found... Create one via compile_bpir`.

The error message is helpful. The tool name is not — agents try this tool first expecting it to do the work. Only after the error do they discover the compile_bpir prerequisite.

**Workaround:** Know the convention. Always run `compile_bpir` with `widget_event` entry first, then call `widget.bind_event` to verify.

**Proposal:** Two options:
1. **Rename** to `widget.verify_event_binding` or `widget.refresh_event_binding` to match behavior.
2. **Extend** to actually create the binding when missing — take `eventName` and the handler node or BPIR body and construct the BndEvt in one call.

## History
- `#1-misleading-name-repro` `OPEN` reporter — First instinct was `widget.bind_event` for creating a button OnClicked handler. Got `BPIR_REQUIRED` pointing to compile_bpir. Would have saved a step to either rename or extend the tool.
- `#2-clarified-summary-error` `IN-REVIEW` developer — Clarified tool Summary (`WidgetEventBindingHandler.cpp:17`) from "Ensures a BndEvt node exists..." to "Verify/refresh an existing widget event binding. Create the BndEvt node first via compile_bpir with 'entry widget_event <WidgetName>.<EventName>() { ... }'". Rewrote the `BPIR_REQUIRED` error message (lines 64-68) to explicitly state "This tool only verifies existing bindings" and show the exact compile_bpir syntax including the blueprint path. No rename (preserves callers), no behavior change.
- `#3-verified-new-error-text` `DONE` tester — Verified via MCP: `widget_bind_event` on `TestButton` (with no BndEvt node) returned the new error text verbatim: `"No BndEvt node for TestButton.OnClicked on /Game/App/UI/Test/W_McpVerifyTemp. This tool only verifies existing bindings. To create one, call compile_bpir with body \"entry widget_event TestButton.OnClicked() { ... }\" first."` Note: the ToolSearch schema catalog may still cache the old Summary string — runtime behavior is correct.
