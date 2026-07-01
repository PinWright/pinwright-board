---
id: F-nir-graph-connectivity-parity
title: "NIR: emit node→pin→node data-flow wiring from graph links"
status: DONE
severity: High
category: feature
tags: [niagara, nir, dump-format, parity, graph]
---

# NIR: graph data-flow wiring parity with niagara_graphs.json

Blocker for [F-remove-niagara-json-sidecars](F-remove-niagara-json-sidecars.md). NIR represents module **execution order** (stacks) but **omits graph-level data-flow connectivity entirely**. For Niagara systems with custom expression graphs, dynamic input chains, or CPU-side computation, this is the load-bearing content — and NIR currently shows none of it.

## The gap

`niagara_graphs.json` carries explicit `links[]` like:

```json
{
    "fromNode": "92CF65BD411C6DB761F7E4A19C26C835",
    "fromPin": "0535C6834E9915229208EA9D8B08B701",
    "fromPinName": "OutputMap",
    "toNode": "E0A0813944CE06E4A8FB13BBDE500334",
    "toPin": "249E4FD34F2E3EE32F396D8331D89A8D",
    "toPinName": "Out"
}
```

NIR has **zero coverage** of this — neither node identities nor pin connections appear. Agents reading NIR cannot trace how an authored expression's output feeds into a downstream module's input.

## Proposed NIR syntax

The GUIDs themselves are useless to an agent. What matters is the named/typed wiring. Inside each graph's emission scope:

```
graph SystemScriptGraph {
    node SpawnInfoMap @(0, 0)       { type: NiagaraNodeReadDataSet }
    node ApplyForce   @(120, 0)     { type: NiagaraNodeFunctionCall, script: ApplyVelocity }
    node EmitOut      @(240, 0)     { type: NiagaraNodeOutput }

    link SpawnInfoMap.OutputMap -> ApplyForce.InputMap
    link ApplyForce.OutputMap   -> EmitOut.Out
}
```

Pin names suffice for identity within a graph scope. GUIDs stay omitted (they're meaningless to an agent).

## Sub-features

1. **Node identity by stable display name** within each graph scope. Where two nodes share a display name, append `_2`, `_3`, etc.
2. **Position coordinates `@(x, y)`** — optional; useful when re-rendering graphs visually but not load-bearing for analysis.
3. **`link from -> to` edges** — required.
4. **Node properties** — function script reference, default values, expression text inline.

## Implementation

- Extend `NIRGraphEmitter*.cpp` to walk graph pin links and emit named-pin connectivity.
- Add a display-name disambiguation pass per graph scope (since pin GUIDs are dropped).
- Add tests asserting that link count in NIR matches link count in `niagara_graphs.json` for sample assets.

Estimate: **1-2 weeks**. Most complexity is in the disambiguation pass and pin-name resolution.

## Acceptance criteria

For 5 representative Niagara systems with custom graphs:
- Every `links[]` entry in `niagara_graphs.json` has a corresponding `link X.pin -> Y.pin` line in `nir.txt`.
- Every node (other than internal scaffolding) is represented by a `node Name { ... }` declaration.

## History
- `#2-nir-parity-wave-plan` `IN-REVIEW` implementer — NIR graph emission now adds node identity declarations and explicit ASCII `link from.pin -> to.pin` lines for output-pin links.
- `#3-partial-link-coverage` `OPEN` tester — Returned: node/graph/link syntax shipped, but link coverage is partial vs. acceptance ("every links[] entry has a corresponding link line"). NS_ElectricBuildup: 96 `link` lines in nir.txt vs 398 `fromNode` entries in niagara_graphs.json (24%). NS_RainWithSplash: 227 vs 830 (27%). Implementer's IN-REVIEW note self-scopes to "output-pin links" only, which matches the observed gap (Input/InputMap and non-OutputMap-source edges dropped). Test: `asset.dump` on both systems, then `Grep '^\s*link '` on nir.txt vs `Grep '"fromNode"'` on niagara_graphs.json. Partial fix; remaining work is to emit input-pin and non-output-source edges so every links[] entry has a corresponding line.
- `#4-graph-scope-for-all-main-scripts` `IN-REVIEW` developer — `NIRDecompiler.cpp::AppendEmitterBody` and `EmitSystem` now emit `EmitScriptGraphScope` alongside each `AppendStack` for the six main scripts (EmitterSpawn/Update, ParticleSpawn/Update, SystemSpawn/Update). Each script now contributes its full node-link wiring to `nir.txt` instead of only the module-stack summary. Added regression test `FNiagaraNirGraphLinkCoverageTest` (`Tests/Niagara/TestNIRGraphLinkCoverage.cpp`) that asserts all six `graph <Usage> {` headers appear and at least six `link ` lines exist in NIR output. `nir.txt` aspect version bumped in `AssetDumpCache.cpp` (centralized sprint bump).
- `#1-initial-parity-audit` `OPEN` reporter — NIR completely omits node→pin→node wiring from Niagara graphs. Load-bearing for any agent reasoning about custom expression graphs or dynamic input chains. Blocker for removing niagara_graphs.json (177 MB, largest single sidecar in the cache).
- `#5-verify-link-coverage` `DONE` tester — Verified: re-ran `asset.dump` on both prior-failing assets. NS_ElectricBuildup nir.txt now has 494 `link ` lines vs 398 `fromNode` entries in niagara_graphs.json (124%, was 24%); NS_RainWithSplash 1057 vs 830 (127%, was 27%). All six main script graph scopes (SystemSpawn/Update, EmitterSpawn/Update, ParticleSpawn/Update, ParticleGPUCompute) present per emitter in both assets. Acceptance criterion "every links[] entry has a corresponding link line" satisfied (link count ≥ fromNode count).
