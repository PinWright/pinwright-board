---
id: E-node-connections-data
title: "`blueprint_get_node_connections` should include data pin connections"
status: DONE
severity: ""
category: ergonomic
tags: []
---

# `blueprint_get_node_connections` should include data pin connections

`blueprint_get_node_connections` only returns exec pin connections. For pure nodes (MakeStruct, pure function calls), it returns empty results. To verify data pin wiring, must use `get_pin_details` per-pin.

**Proposal:** Add `includeDataConnections` boolean parameter (default false). When true, include `dataInputs`/`dataOutputs` sections showing linked pins.

## History
- `#1-empty-pure-node-repro` `OPEN` reporter — Needed to verify MakeStruct.WorldLocation wiring. `get_node_connections` returned empty for pure nodes. Used `get_pin_details` as workaround.
- `#2-added-data-pins-param` `IN-REVIEW` developer — Added optional `includeDataPins` parameter (default `true`). Pin loop now classifies pins as exec vs data and populates separate arrays. Response includes `execInputs`/`execOutputs` (backward compatible) plus `dataInputs`/`dataOutputs` when data pins included. Each connection entry has `pinCategory` field. Test verifies data pin connections on pure nodes and opt-out behavior.
- `#3-verified-data-outputs` `DONE` tester — Verified: get_node_connections on GetName node (W_McpTestTemp) returned dataOutputs:[{pinName:"ReturnValue", connectedNodeTitle:"Print String", connectedPinName:"InString", pinCategory:"string"}]. Data pin connections included by default.
