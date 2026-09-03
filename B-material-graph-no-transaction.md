---
id: B-material-graph-no-transaction
title: "All material.graph mutators run without an FScopedTransaction, so editor.undo cannot restore their changes"
status: OPEN
severity: Medium
category: bug
tags: [material, graph, undo, transaction, authoring]
---

# Seven material graph write verbs have no transaction

`MaterialGraphHandler.cpp` registers seven mutators: `add_node`, `remove_node`, `connect_nodes`,
`break_connections`, `add_texture_sample`, `add_expression`, and `create_nodes`. The file contains
no `FScopedTransaction`. Three later creation paths call `Modify()`, while the first four rely on
direct collection/input changes and `PostEditChange`; none has an active transaction in which an
undo record could be stored.

The result is most costly for `remove_node`: losing a configured expression cannot be recovered by
Undo. Connections and batch-created nodes likewise have to be found and repaired manually. UE's
Material Editor deletion path opens an `FScopedTransaction` and modifies/breaks nodes before
removal (`MaterialEditor.cpp:6245-6249`, `:6340-6402`), providing the local engine pattern.

## What it should do

Open one scoped transaction per RPC and call `Modify()` on the material/function plus objects whose
serialized fields change. A batch create should remain one undo step. Add Undo/Redo coverage for a
wire, a created node, and a configured node deletion.

## Workaround

Save/revision-control the asset before mutation and revert the package when a graph edit must be
undone.

## Related

- `B-material-remove-incomplete`
- `B-pcg-graph-mutators-no-transaction`

## History
- `#1-source-scan` `OPEN` reporter -- Source-only census of all registrations and transaction/
  `Modify` calls in the handler; no Undo execution under the scan constraints.
