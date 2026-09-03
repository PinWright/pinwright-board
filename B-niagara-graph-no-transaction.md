---
id: B-niagara-graph-no-transaction
title: "niagara.graph.remove_node mutates outside any editor transaction"
status: OPEN
severity: Medium
category: bug
tags: [niagara, graph, undo, transaction, remove-node]
---

# Node removal cannot be reversed with editor.undo

`NiagaraGraphHandler.cpp` has three structural graph mutators. `create_node` opens an
`FScopedTransaction` and calls `Graph->Modify()` (`:744-745`). `connect_pins` delegates to UE 5.8's
`UEdGraphSchema_Niagara::TryCreateConnection`, which opens its own scoped transaction
(`EdGraphSchema_Niagara.cpp:1452-1455`). `remove_node`, however, calls
`TargetGraph->RemoveNode` (`NiagaraGraphHandler.cpp:414`) with no transaction in the handler or the
dispatcher.

UEdGraph::RemoveNode calls `Modify()` (`EdGraph.cpp:261-264`), but Unreal records undo data only
while a transaction is active. The deletion therefore cannot be reversed through the plugin's
normal `editor.undo` path; the Niagara node and its incident connections must be reconstructed
manually or the asset reverted.

## What it should do

Wrap the removal in a scoped transaction and add an undo/redo test that proves graph identity and
links before, after, after Undo, and after Redo. Keep `connect_pins` on the engine-owned transaction
rather than nesting another one.

## Workaround

Save/revision-control the asset before these calls and use GUIDs to reduce wrong edits.

## Related

- `B-niagara-graph-target-fallback`
- `B-niagara-connect-pins-sends-no-graph-notification`

## History
- `#1-source-scan` `OPEN` reporter -- Source-only transaction census, including the engine
  implementations behind both calls; editor.undo was not run due to the scan constraints.
