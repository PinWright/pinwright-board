---
id: E-pin-details-batch-requests-shape-mismatch
title: "get_pin_details_batch uses a different param shape (requests:[{nodeId,pinName}]) than its sibling get_node_details_batch (nodeIds:[]), and its optional pinName isn't discoverable"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint-graph, get_pin_details_batch, get_node_details_batch, batch, param-shape, misuse-then-correct]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `get_pin_details_batch` param shape diverges from `get_node_details_batch`, and its optional `pinName` is invisible — agents misuse-then-correct

The two sibling batch-inspection RPCs in `blueprint.graph.*` take **different
top-level array params for the same conceptual input** (a set of nodes):

- `blueprint.graph.get_node_details_batch` → `nodeIds: ["<id>", ...]` (array of node-id strings).
- `blueprint.graph.get_pin_details_batch` → `requests: [{nodeId, pinName}, ...]` (array of objects).

An agent that has just used (or is about to use) `get_node_details_batch` and
then reaches for `get_pin_details_batch` naturally carries the `nodeIds` shape
over. The result is `[MISSING_REQUIRED_PARAM] Missing required parameter
'requests' (type: array)` — a hard error that forces a re-read of the schema
and a corrected call (or, as this task did, a pivot away from the method
entirely). The two methods are adjacent in the namespace, do adjacent jobs
(node details vs pin details), and differ only because one wires a per-row
`pinName` — yet their entry-point param names share nothing.

Compounding this: in the handler, **`pinName` is optional per request** — when
omitted, `get_pin_details_batch` reports *all* pins on the node
(`BlueprintGraphInspectionHandler.cpp`: `if (!PinName.IsEmpty()) {...single pin...}
else { PinsToReport = TargetNode->Pins; }`). That optional-pinName path is
exactly the "I don't know the pin names yet, give me the whole pin shape" case —
the most common reason an agent calls a pin-inspection method before its first
`connect_pins`. But the param is documented as `Array of {nodeId, pinName}
objects` with no hint that `pinName` may be dropped, so an agent reads it as
"I must already know the pin name to ask about its pins" — a chicken-and-egg
that drives it to the other method instead. The method that should have served
the intent in one call looks unusable for it.

## What it should do

Two independent, small ergonomic fixes (either alone removes most of the friction):

1. **Accept `nodeIds` as an alias on `get_pin_details_batch`** (treat
   `nodeIds: ["<id>", ...]` as `requests: [{nodeId}, ...]` with no pinName →
   all pins), so the two batch methods are call-compatible for the common
   "dump every pin on these nodes" case and a carried-over `nodeIds` arg just
   works. (Legacy-alias precedent already exists in this namespace — see the
   `target`/`variableName`/`memberName` aliases in `blueprint.graph.create_node`.)
2. **Document `pinName` as optional** in the `requests` param description
   (`Array of {nodeId, pinName?} — omit pinName to return all pins on the node`)
   and surface that in the `docs/wiki-src/blueprint.graph.md` overlay so agents
   know the whole-node pin dump is a one-call operation and don't pivot away.

A lighter-touch alternative to (1): make the `MISSING_REQUIRED_PARAM` error on
this method name the sibling shape explicitly ("expected `requests:
[{nodeId, pinName?}]`; if you meant whole-node pin dumps, pass
`requests:[{nodeId}]`") so the corrective call is obvious from the error alone.

This is an ergonomic/param-shape + docs improvement, not a code-behavior bug —
the method works; the awkward part is its discoverability and its divergence
from its sibling.

## Evidence

mcp-fuzz task on `blueprint.graph` (hand-wiring `BP_RegenActor`'s Tick regen
graph, 32 calls, outcome clean/tool_bug). The agent had just created six nodes
and wanted their pin shapes before connecting. Call log:

- `blueprint.graph.get_pin_details_batch` — *"nodeIds batch (no requests
  param)"* → `is_error:true`, `error_text:"[MISSING_REQUIRED_PARAM] Missing
  required parameter 'requests' (type: array)"`.
- next call `blueprint.graph.get_node_details_batch` — *"6 node ids; confirmed
  pin names A/B/Value/Min/Max/execute/Health"* → `ok`.

Friction note (verbatim): *"get_pin_details_batch rejected my nodeIds arg
wanting a {nodeId,pinName} 'requests' array (a chicken-and-egg since I needed
pin names); I pivoted to get_node_details_batch which took nodeIds and gave
full pin shapes."* This is a textbook misuse-then-correct: a sibling method's
param shape (`nodeIds`) carried into a divergently-named param (`requests`),
one wasted error round-trip, and an abandoned method whose optional-`pinName`
path would actually have answered the agent's intent.

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of the `blueprint.graph` BP_RegenActor task (32 calls, outcome clean): `get_pin_details_batch` requires `requests:[{nodeId,pinName}]` while its sibling `get_node_details_batch` requires `nodeIds:[]`; the agent carried the `nodeIds` shape over, hit `[MISSING_REQUIRED_PARAM] Missing required parameter 'requests'`, and pivoted to `get_node_details_batch` (1 wasted error call). Per-request `pinName` is in fact optional (handler dumps all node pins when omitted) — the exact "I don't know pin names yet" intent — but that isn't documented, so the method looks like a chicken-and-egg and gets abandoned. Propose: accept a `nodeIds` alias on `get_pin_details_batch` (per the namespace's existing legacy-alias precedent), document `pinName` as optional in the param + `docs/wiki-src/blueprint.graph.md` overlay, and/or have the MISSING_REQUIRED_PARAM error name the expected `requests:[{nodeId}]` shape. Distinct from the judge's `B-add-variable-default-value-ignored` (CDO default bug) and from `E-graph-standard-exec-pin-names` / `E-blueprintgraph-handler-split` (which only mention these methods, not the param-shape divergence).
