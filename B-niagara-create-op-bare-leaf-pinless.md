---
id: B-niagara-create-op-bare-leaf-pinless
title: "niagara.graph.create_node accepts search_ops' bare opName ('Mul') but silently creates a pinless 'Unknown' op node"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, niagara-graph, create-node, op-registry, silent-noop]
---

# `create_node` op node is silently broken when given the bare leaf opName

`niagara.graph.search_ops` returns each op as `{ opName, signature, category, score }`.
For multiply it returns `{"opName":"Mul","signature":"Numeric::Mul",...}`. The
`opName` field is the bare leaf name; the `signature` field is the
fully-qualified registry key.

`niagara.graph.create_node`'s payload field is also called `opName`. Passing the
**bare leaf** value that `search_ops` puts in its `opName` field
(`payload.opName="Mul"`) returns a **clean success** (`ok:true`, no error code)
but produces a structurally-broken op node: `pins: []`. In the Niagara editor
the node shows up titled `Unknown` with no A/B/Result pins, so it cannot be
wired and silently breaks any graph splice that relies on it. The caller only
discovers the breakage by re-reading the graph and noticing the node has no
pins.

Passing the **registry key** from the `signature` field
(`payload.opName="Numeric::Mul"`) works and yields the proper pins. So the one
field a caller would most naturally feed forward from `search_ops` (its
`opName`) is exactly the form that silently fails, while the form that works
(`signature`) is undocumented as the required input.

This is a silent success-with-broken-effect: a node is created and reported OK,
but it is non-functional. It is the same failure class as
[`F-graph-create-node-timeline`](F-graph-create-node-timeline.md) (factory
emitted a pinless/unbacked node) but in the Niagara op path.

## Root cause

`NiagaraGraphHandler.cpp` (`ApplyCreateNodePayload`, op branch ~line 493):
the validation strips any `Category::` prefix and matches the **leaf** against
`KnownOpLeafNames`, so the bare `"Mul"` passes validation — but the handler then
stores the **un-canonicalized** payload string verbatim:

```cpp
OpNode->OpName = FName(*PayloadOpName);   // stores "Mul", not "Numeric::Mul"
```

The in-file comment already notes "the registry keys are `Category::Leaf`, not
bare `Leaf`", yet the assignment uses the raw `PayloadOpName`. When the op
registry is later consulted for pin allocation, the bare `"Mul"` key resolves to
nothing, so the node gets zero pins and an `Unknown` title.

The fix is to canonicalize: when the input was a bare leaf, store the matched
`Category::Leaf` registry key (the same string `search_ops` reports in its
`signature` field), not the raw input. Either accept the leaf and reconstruct
the full key, or reject the bare leaf with a clear error pointing at the
`signature` form. (The DONE verification `#5-verify-create-node-op` of
`F-niagara-graph-create-node` used `opName:"Add"` and only checked the returned
`nodeId`/`nodeClass`, never the pins — which is why this slipped through.)

**Workaround:** pass the `signature` value from `search_ops`
(`payload.opName="Numeric::Mul"`), not the `opName` value, when creating a
`NiagaraNodeOp`.

**Fix:** canonicalize `OpNode->OpName` to the full `Category::Leaf` registry key
inside the op branch of `ApplyCreateNodePayload` (and/or document that
`payload.opName` requires the `signature` form); add a regression test that
creates an op node with the bare leaf and asserts the result has non-empty pins.

## Repro (verbatim, live)

- `niagara.graph.search_ops` `{query:"mul"}` →
  `{"opName":"Mul","signature":"Numeric::Mul","category":"Numeric","score":1000}`
- `niagara.graph.create_node`
  `{assetPath:"/Game/ExampleContent/EnhancedInput/VFX/Confetti/NS_Confetti",
    target:{kind:"graph",emitter:"ConfettiBurst",scriptUsage:"ParticleUpdate"},
    nodeClass:"NiagaraNodeOp", x:400, y:200, payload:{opName:"Mul"}}` →
  `{"nodeId":"0BB3E6FE...","nodeClass":"/Script/NiagaraEditor.NiagaraNodeOp","pins":[],"compiled":false,"saved":false}`
  — success, but **`pins:[]`** (broken).
- Same call with `payload:{opName:"Numeric::Mul"}` →
  `pins:[{A,input},{B,input},{Result,output},{Add,input}]` (correct).

## History
- `#0-process-audit-second-occurrence` `OPEN` reporter — Cross-task aggregation: the `niagara.graph` end-to-end op-splice task (38 calls) hit this again as live process friction. The agent fed `search_ops`' `opName` field verbatim (`payload.opName="Mul"`) to `create_node`, got `ok:true`, then a re-read showed `title="Unknown"` with 0 pins; it had to `remove_node` the broken node and recreate with `payload.opName="Numeric::Mul"` (a create→reread→remove→recreate trial-and-error cycle). Friction note verbatim: "create_node with payload opName=\"Mul\" (the short name search_ops returns) reported success but produced a pinless title=\"Unknown\" node — only the full signature \"Numeric::Mul\" works, and nothing in the wiki/search_ops/create_node docs says the payload needs the Category::Op form." Confirms severity and that the docs half of the proposed fix (document that `payload.opName` requires the `signature` form) is load-bearing — independent agents keep feeding the wrong field forward.
- `#2-fix-canonicalize-op-name` `IN-REVIEW` developer — Root-cause fix: the op branch of `ApplyCreateNodePayload` now canonicalizes a bare leaf opName to the full `Category::Leaf` engine registry key before storing it, so feeding `search_ops`' `opName` field forward (`payload.opName="Mul"`) resolves to `Numeric::Mul` and the node allocates its real A/B/Result pins instead of a pinless `Unknown`. Unified the two divergent hand-mirrored op snapshots (the leaf-only `TSet` in `NiagaraGraphHandler.cpp` and the `{Leaf, Category}` table in `NiagaraSearchHandler.cpp`) into one shared source of truth — new header `Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/NiagaraOpCatalog.h` — which both `create_node` (validate + canonicalize) and `search_ops` (results + signature) now consume, guaranteeing the search_ops→create_node round-trip. An uncatalogued op still returns `INVALID_OP`. Files: `NiagaraOpCatalog.h` (new), `Handlers/Niagara/NiagaraGraphHandler.cpp`, `Handlers/Niagara/NiagaraSearchHandler.cpp`, `docs/wiki-src/niagara.graph.md` (documents that `payload.opName` accepts either the `opName` or `signature` form and is auto-canonicalized). Regression test: `Tests/Niagara/TestNiagaraGraphCreateNode.cpp` case 1 now asserts bare `"Add"` is stored as `Numeric::Add` (was asserting the buggy verbatim `"Add"`), plus new cases 1b/1c asserting bare `"Mul"`→`Numeric::Mul` and that the full key `"Numeric::Mul"` is accepted unchanged — case 1 would fail if the canonicalization were reverted. Not compiled/tested here; a later phase drives it green.
- `#1-initial-repro` `OPEN` reporter — Replayed live against NS_Confetti emitter ConfettiBurst ParticleUpdate. `payload.opName="Mul"` (the bare leaf `search_ops` returns in its `opName` field) returns clean success with `pins:[]` (titled `Unknown`, non-functional); `payload.opName="Numeric::Mul"` (the `signature` field) returns the proper A/B/Result pins. Root cause: op branch of `ApplyCreateNodePayload` in `NiagaraGraphHandler.cpp` validates the stripped leaf against `KnownOpLeafNames` but assigns `OpNode->OpName = FName(*PayloadOpName)` with the un-canonicalized raw string, so a bare leaf never matches a `Category::Leaf` registry key and the node allocates zero pins. The DONE create_node verification used `opName:"Add"` and only checked nodeId/nodeClass, never pins, so the silent breakage was not caught. Not a duplicate: `F-niagara-graph-create-node`/`F-search-api-niagara-graph-nodes` are DONE feature tickets; no existing bug ticket covers the bare-leaf pinless op node.
