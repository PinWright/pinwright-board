---
id: F-search-api-niagara-graph-nodes
title: "No catalog / search for Niagara graph node types (UNiagaraNode*)"
status: DONE
severity: High
category: feature
tags: [search, niagara, niagara-graph, node-discovery]
---

# No catalog / search for Niagara graph node types (UNiagaraNode*)

Inside a `UNiagaraScript`, the graph is built from `UNiagaraNode*`
subclasses — `UNiagaraNodeFunctionCall`, `UNiagaraNodeOp`,
`UNiagaraNodeReadDataSet`, `UNiagaraNodeWriteDataSet`,
`UNiagaraNodeIf`, `UNiagaraNodeOutput`, `UNiagaraNodeInput`,
`UNiagaraNodeCustomHlsl`, and ~30 others. The story is the same
K2Node-style two-tier shape as Blueprint graphs:

- `UNiagaraNodeFunctionCall` wraps a `UNiagaraScript` asset (which
  function script to invoke).
- `UNiagaraNodeOp` wraps an `FName` op id from the Niagara op registry
  (`Add`, `Multiply`, `Sin`, `Vector3Cross`, `Saturate`, etc.).
- `UNiagaraNodeReadDataSet` / `WriteDataSet` wrap a dataset reference.
- Container-only nodes (`If`, `Input`, `Output`, `Reroute`) have no
  payload.

There is no RPC that enumerates either the `UNiagaraNode*` taxonomy or
the op registry. `asset.dump` / `niagara.inspect` reads the existing
graph nodes from an asset, but doesn't expose the *catalog* of what
could be added.

**Status note:** Priority is **Low** because most Niagara authoring
happens at the **stack module** layer ([`F-search-api-niagara-modules`])
rather than inside raw `UNiagaraScript` graphs. The stack is the
designer-facing surface; the script graph is reserved for module
authoring (which is rarer and usually best done in the Niagara editor
UI). Agents that compose stacks rather than write modules will not need
this catalog; agents that author modules will.

**Use cases blocked (when Niagara module authoring is in scope):**

1. "What ops are available for `UNiagaraNodeOp`?" — Niagara's op
   registry is independent of UE's math expression set; common names
   line up (Add, Multiply) but the registry contains Niagara-specific
   ops (e.g. SIMD-friendly variants, GPU-only ops).
2. "What graph node classes exist?" — `UNiagaraNode*` is small enough
   to enumerate manually, but the generic class search
   ([`F-search-api-native-uclasses`]) covers this case fine with
   `parentClass="NiagaraNode"`.

**Proposal:** Two RPCs mirroring the Blueprint pair:

```
niagara.graph.list_node_types() -> {
    nodeTypes: [{
        className: "NiagaraNodeFunctionCall",
        displayName: "Function Call",
        payloadKind: "script_asset",   // "script_asset" | "op_name" | "dataset_ref" | "none"
        category: "Function"
    }, ...]
}

niagara.graph.search_ops(query?: string, limit?: number) -> {
    results: [{
        opName: "Add",
        signature: "Add(float, float) -> float",
        category: "Math",
        gpuOnly: false,
        score: number
    }, ...],
    totalMatches: number
}
```

`list_node_types` is the K2Node-style container catalog (small, ~30
classes). `search_ops` covers the `UNiagaraNodeOp` payload registry —
this is the larger surface, since Niagara's op set is comparable in
size to the math-expression catalog.

**Cross-ref:** Strongly paired with
[`F-search-api-niagara-modules`](F-search-api-niagara-modules.md). If
that ticket lands and module-stack authoring becomes the only Niagara
authoring path, this ticket can stay deferred indefinitely. File it
now so the gap is documented; revisit if/when an agent workflow
actually reaches into raw `UNiagaraScript` graphs.

## History
- `#1-no-niagara-graph-catalog` `OPEN` reporter — Filed for completeness alongside the other discovery-gap tickets. `UNiagaraNode*` is K2Node-style two-tier (container + payload), where the payload is either a script asset (`FunctionCall`), an op name from the Niagara op registry (`Op`), a dataset ref, or nothing. The generic UClass search covers the container catalog; the op registry needs a dedicated RPC since it's a Niagara-internal `FName` registry, not a UClass set. Priority Low because most authoring happens at the stack-module layer rather than inside raw script graphs. Proposes `niagara.graph.list_node_types` (containers) and `niagara.graph.search_ops` (payload registry for `UNiagaraNodeOp`), mirroring the Blueprint `list_node_types` + `search_api` pair.
- `#2-reviewed-and-confirmed` `OPEN` tester — Verified against source under `Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/`: no `niagara.graph.list_node_types`, no `niagara.graph.search_ops`, no op-name catalog RPC of any kind. Only mutating graph RPCs are `add_module`, `connect_pins`, `disconnect_pin`, `set_pin_default`, `set_parameter`, `remove_node`. No duplicate niagara-graph-catalog tickets in `docs/board/`. Severity elevated **Low → High**: the original Low rating was gated on raw `UNiagaraScript` graph authoring being out of scope, but sibling ticket `F-niagara-graph-create-node` (Critical) establishes that graph authoring IS the only path to new modules / dynamic inputs / function scripts since Niagara has no text IR. Without `search_ops`, even a working `create_node` is unusable for `UNiagaraNodeOp` — the largest payload class — because the caller has no way to pick a valid op `FName`. The two tickets should remain separate (mirrors the Blueprint `list_node_types`/`search_api` vs `create_node` split) but should land together; `search_ops` is the gating dependency for any `UNiagaraNodeOp` creation. `list_node_types` is partially substitutable by generic UClass search with `parentClass="NiagaraNode"` per the ticket's own note, so the High (not Critical) rating reflects that `search_ops` is the truly load-bearing half.
- `#3-implemented-search-handlers` `IN-REVIEW` developer — New `NiagaraSearchHandler.cpp` registers `niagara.graph.list_node_types` (TObjectIterator over UNiagaraNode subclasses, payloadKind from a static table) and `niagara.graph.search_ops` (FNiagaraOpInfo::GetOpInfoArray + score-rank). Dropped `gpuOnly` from search_ops response — no such field on FNiagaraOpInfo. Helpers exposed via NiagaraSearchHandler.h for test reach.
- `#4-fix-review-issues` `IN-REVIEW` developer — Added four behavioral dispatcher tests (ListNodeTypes payload-kind assertions, SearchOps FindsAdd/EmptyReturnsAll/GarbageReturnsEmpty) via InvokeHandlerWithCapture; pre-stringified op names in FScoredOp to eliminate O(N log N) FName::ToString() allocations in sort comparators.
- `#5-verify-search-handlers` `DONE` tester — Verified: `niagara.graph.list_node_types` returns Niagara node classes with payload kinds including `NiagaraNodeFunctionCall=script_asset`, `NiagaraNodeOp=op_name`, and dataset nodes as `dataset_ref`; `niagara.graph.search_ops` with `query="Add", limit=10` returns `Add` with `totalMatches=2`, while a garbage query returns `totalMatches=0`.
