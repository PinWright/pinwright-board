---
id: F-batch-pin-defaults
title: "No batch `blueprint_graph_set_pin_default_value`"
status: DONE
severity: ""
category: feature
tags: []
---

# No batch `blueprint_graph_set_pin_default_value`

14 enum pins required 14 sequential MCP calls. Batch endpoint accepting `[{nodeId, pinName, value}, ...]` would reduce to 1 call.

## History
- `#1-sequential-pin-calls` `OPEN` reporter — 12 ESlateVisibility + 2 EDroneArmState pins fixed one by one.
- `#2-added-batch-handler` `IN-REVIEW` developer — Added `blueprint.graph.set_pin_default_values` (plural) handler. Extracted `ApplyPinDefaultValueCore` helper from singular handler (DRY). Batch handler loads BP once, opens single `FScopedTransaction`, iterates `updates` array calling helper per item. Continues on individual failures. Response includes `results` array with per-item status, plus `totalUpdates`/`successCount`/`failureCount`. Two tests: batch success and partial failure.
- `#3-verified-batch-response` `DONE` tester — Verified: called blueprint.graph.set_pin_default_values with 2 updates (InString="batch1", Duration="5.0") on W_McpTestTemp PrintString node. Response: totalUpdates:2, successCount:2, failureCount:0. Per-item results include requestedValue/value confirmation.
