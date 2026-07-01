---
id: E-node-comment-field-name-drift
title: "blueprint.graph node comment field is 'comment' in get_nodes but 'nodeComment' in get_node_details"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, graph, response-field, naming]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# blueprint.graph node comment field is 'comment' in get_nodes but 'nodeComment' in get_node_details

The same logical field — a graph node's comment string — is returned under two
different JSON keys by sibling read RPCs in the `blueprint.graph.*` namespace:

- `blueprint.graph.get_nodes` returns it as **`comment`**.
- `blueprint.graph.get_node_details` returns it as **`nodeComment`**.

A list-then-detail workflow (the natural pattern: `get_nodes` to enumerate, then
`get_node_details` to drill in) keys off `comment` from the list path and finds
no such field in the detail path — the value is there, just spelled
differently. The write side adds a third spelling: `set_node_property` takes
`propertyName: "Comment"` (capitalized) for the same field. So one round-trip
(enumerate → inspect → write → re-inspect) touches three spellings of one
field: `comment`, `nodeComment`, and the `Comment` property name.

Nothing fails and no data is wrong — both reads report the identical value — so
this is pure friction, not a bug. But a caller that builds a dict keyed on the
list-path field name, or copies the readback shape from one RPC to the other,
silently drops the comment. Standardizing the response key (or documenting the
divergence on the wiki page) would remove the snag.

**Workaround:** Read the comment as `comment` from `get_nodes` and as
`nodeComment` from `get_node_details`; don't assume the two read RPCs share the
field name.

**Fix:** Pick one canonical response key for the node comment across both read
RPCs (most other node fields — `nodeId`, `nodeName`, `nodeType`, `nodeTitle`,
`x`, `y` — already match between the two methods; only the comment diverges).
Aliasing `comment` into `get_node_details` (keeping `nodeComment` for back-compat)
or renaming to match `get_nodes` would align them.

## History
- `#1-initial-repro` `OPEN` reporter — Hit during a BP event-graph tidy-up (set a node's Comment/X/Y via `blueprint.graph.set_node_property`, then read back). Verbatim, on node `6A79A5724CA8E46FF51D7299BF073CE1` ("Toggle light" CustomEvent) of `/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic`: `blueprint.graph.get_nodes {graphName:"EventGraph"}` → `"comment":"Entry point: initialize bulb state on begin play"`; `blueprint.graph.get_node_details {graphName:"EventGraph", nodeId:"6A79A5724CA8E46FF51D7299BF073CE1"}` → `"nodeComment":"Entry point: initialize bulb state on begin play"` — same value, two keys. Write side uses a third spelling: `set_node_property {propertyName:"Comment", value:"…"}`. Every other node field (`nodeId`/`nodeName`/`nodeType`/`nodeTitle`/`x`/`y`) is spelled identically across the two reads; only the comment diverges. Replay-confirmed live; the set_node_property writes themselves all round-tripped correctly (x=320, y=160, comment stuck).
