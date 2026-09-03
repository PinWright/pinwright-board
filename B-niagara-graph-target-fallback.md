---
id: B-niagara-graph-target-fallback
title: "Niagara graph mutators silently target Spawn for invalid script types and the last duplicate-title node"
status: OPEN
severity: High
category: bug
tags: [niagara, graph, wrong-target, connect-pins, remove-node, validation]
---

# Target resolution is permissive in two destructive dimensions

The shared `ResolveNiagaraGraph` selects Spawn first and changes to Update only when
`scriptType == "Update"` exactly (`NiagaraGraphHandler.cpp:47-81`). Any other non-empty value --
including case drift or a documented-stage spelling used elsewhere -- silently targets Spawn.
`connect_pins` and `remove_node` both use this resolver and then return success for that graph.

Inside `connect_pins`, a node may match by GUID, UObject name, or display title. The scan does not
stop or count matches; each later match overwrites the earlier pointer (`:282-293`). Duplicate
Niagara node titles are normal in real graphs, so a title resolves to the last matching node in
array order. The response then echoes the request strings rather than the resolved node GUIDs
(`:341-348`), hiding which nodes were actually wired.

These paths can edit a valid but unintended graph/node and report success. The GUID-only
`remove_node` lookup demonstrates the safe identity shape already in the same file.

## What it should do

Parse `scriptType` case-insensitively against an explicit Spawn/Update allow-list and reject
unknown values. Resolve node GUIDs first; if name/title support remains, require exactly one match
and return `AMBIGUOUS_NODE`. Include resolved graph usage and node GUIDs in the success response.

## Workaround

Pass exact `Spawn` or `Update` and use node GUIDs, never display titles.

## Related

- `B-niagara-connect-pins-sends-no-graph-notification`

## History
- `#1-source-scan` `OPEN` reporter -- Both wrong-target branches and the request-echo response were
  confirmed in current source; no asset mutation was performed.
