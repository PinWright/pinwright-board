---
id: F-niagara-graph-create-node
title: "Cannot create non-module graph nodes (UNiagaraNodeOp/If/Input/Output/CustomHlsl/...) imperatively"
status: DONE
severity: Critical
category: feature
tags: [niagara, niagara-graph, authoring, script]
---

# Cannot create non-module graph nodes imperatively

`niagara.graph` exposes only four mutating RPCs against a script graph:

- `niagara.graph.add_module` (legacy) / `niagara.add_module` — adds a
  `UNiagaraNodeFunctionCall` referencing an existing `UNiagaraScript`
  asset.
- `niagara.connect_pin` / `niagara.disconnect_pin`
- `niagara.set_pin_default` (and `niagara.graph.set_parameter`)
- `niagara.graph.remove_node`

There is **no equivalent of `blueprint.graph.create_node`** for the
~30 other `UNiagaraNode*` subclasses: `UNiagaraNodeOp`,
`UNiagaraNodeIf`, `UNiagaraNodeInput`, `UNiagaraNodeOutput`,
`UNiagaraNodeReadDataSet`, `UNiagaraNodeWriteDataSet`,
`UNiagaraNodeCustomHlsl`, `UNiagaraNodeReroute`, `UNiagaraNodeUsageSelector`,
`UNiagaraNodeStaticSwitch`, `UNiagaraNodeConvert`, etc.
Confirmed by `call("niagara.graph.add_node")` returning NOT FOUND.

## Why it matters

The stack-module layer (`niagara.add_module`) is sufficient for
**consuming** existing modules and dynamic inputs. It is **not**
sufficient for **authoring** them. A `UNiagaraScript` module asset is
itself a graph built from `UNiagaraNode*` subclasses. To create a new
module that adds curl noise, computes a forward vector, or branches on
a static switch, the agent must either:

1. Open the Niagara editor UI and click-author the graph (defeats the
   point of MCP).
2. Duplicate a similar existing module and patch it via `set_property`
   per-node (fragile; subject to layout/identity drift; relies on
   `find_nodes` which doesn't exist for Niagara graphs either).
3. `python.execute` calling `UNiagaraGraph::AddNode` directly, bypassing
   the typed-RPC surface.

Niagara has **no text IR** (unlike Blueprint/BPIR, AnimBP/AGIR, and
Material/MGIR), so the imperative surface is the only authoring path.
Until `niagara.graph.create_node` exists, module / dynamic-input / function
script authoring is effectively MCP-out-of-scope.

## Op-registry payload (`UNiagaraNodeOp`)

`UNiagaraNodeOp` wraps an `FName` op id from Niagara's op registry
(`Add`, `Multiply`, `VectorCross`, `Saturate`, `SimpleAddVector`, …).
Filed separately as
[`F-search-api-niagara-graph-nodes`](F-search-api-niagara-graph-nodes.md)
for the discovery half; this ticket is about the *create* half. The two
are paired: a create RPC needs the op id to populate, and an op-registry
search RPC needs a create RPC to feed.

## Proposal

Add a generic `niagara.graph.create_node` mirroring the K2Node-style
shape of `blueprint.graph.create_node`:

```
niagara.graph.create_node(
    assetPath: string,
    target: {                   // graph descriptor (existing shape)
        kind: "system"|"emitter"|"script",
        emitter?: string,
        scriptUsage?: string    // ParticleSpawn / ParticleUpdate / Module / DynamicInput / ...
    },
    nodeClass: string,          // "NiagaraNodeOp" | "NiagaraNodeIf" | "NiagaraNodeInput" | ... | full /Script/NiagaraEditor.* path
    x: number,                  // required, no auto-layout fallback (matches blueprint convention)
    y: number,
    payload?: {                 // node-class-specific; subset documented per nodeClass
        opName?: string,        // NiagaraNodeOp
        inputName?: string,     // NiagaraNodeInput
        inputType?: string,     // NiagaraNodeInput
        customHlsl?: string,    // NiagaraNodeCustomHlsl
        staticSwitchType?: string,
        ...
    },
    compile?: boolean,
    save?: boolean
) -> { nodeId: string, pins: [...] }
```

Plus a small set of convenience constructors for the most common cases
(matching the blueprint pattern of having both `create_node` and
narrower verbs like `create_reroute_node`):

```
niagara.graph.create_reroute_node(assetPath, target, x, y, pinType?) -> {nodeId}
niagara.graph.create_op_node(assetPath, target, opName, x, y) -> {nodeId, pins}
niagara.graph.create_if_node(assetPath, target, x, y, valueType) -> {nodeId, pins}
```

Implementation surface: `UNiagaraGraph::CreateNode` is exposed on the
graph, with the same `UEdGraph`-style API used by `K2Node` creation.
The hard part is post-spawn `AllocateDefaultPins` + payload assignment
for `UNiagaraNodeOp::OpName`, `UNiagaraNodeInput::Input`, etc.
Validate that the resulting node passes
`UNiagaraGraph::ValidateBuiltData` before returning success.

## Workarounds today

- `python.execute` calling `UNiagaraGraph::AddNode<UNiagaraNodeOp>(...)`
  then setting `OpName` directly. Bypasses MCP typing, error codes, and
  the read-first workflow.
- Duplicate an existing module asset and edit it with `set_property` per
  node — works only for modules that already have the desired shape.

## Cross-ref

- `F-search-api-niagara-graph-nodes` — the discovery half (op-registry
  catalog + `UNiagaraNode*` taxonomy). Paired ticket.
- `F-niagara-event-handler-simstage-authoring` (DONE) — established the
  pattern of session-scoped view-model cache and vendored engine
  helpers for Niagara authoring.
- `blueprint.graph.create_node` — reference shape and parameter
  conventions.

## History
- `#1-no-graph-create-node` `OPEN` reporter — Confirmed `niagara.graph` exposes only `add_module` (function-call wrapper), `connect_pins`, `set_pin_default`, and `remove_node`. No generic node-create RPC exists for any of the ~30 `UNiagaraNode*` subclasses. Since Niagara has no text IR, this is the only authoring path for new modules / dynamic inputs / function scripts. Proposes `niagara.graph.create_node` with `nodeClass` + `payload` plus convenience constructors for `Op`, `Reroute`, `If`. Mandatory x/y per the project's positional convention. Paired with the existing discovery-gap ticket `F-search-api-niagara-graph-nodes`.
- `#2-reviewed-and-confirmed` `OPEN` reporter — Verified against source: `NiagaraGraphHandler.cpp:239` hardcodes `NewObject<UNiagaraNodeFunctionCall>` and requires a `UNiagaraScript` module-asset target; no other mutating graph-create RPC exists in `Handlers/Niagara/`. Severity Critical is justified — without this, no `UNiagaraNodeOp` / `If` / `Input` / `Output` / `CustomHlsl` / `Reroute` / `StaticSwitch` can be authored, which means greenfield module / dynamic-input / function-script authoring is fully blocked (only existing-script consumption works via `niagara.add_module`). API shape good: `nodeClass` accepting both short names and full `/Script/NiagaraEditor.*` paths is consistent with the rest of the plugin; `payload` discriminator pattern mirrors how `blueprint.graph.create_node` handles K2Node-specific data. Pairing with `F-search-api-niagara-graph-nodes` is required — without the op-registry / `UNiagaraNode*` catalog, agents can't know which `opName` / `nodeClass` strings are valid to pass. Both tickets must ship together to be usable. No duplicate board entries.
- `#3-implemented-create-node` `IN-REVIEW` developer — Added `niagara.graph.create_node` in `NiagaraGraphHandler.cpp` with per-class payload routing for v1 set: Op, Input, Output, CustomHlsl, StaticSwitch, If, Reroute, Convert (others return UNSUPPORTED_NODE_CLASS). Payload-application helper extracted to `NiagaraGraphCreateNodePayload.h` for test access. Tests in `TestNiagaraGraphCreateNode.cpp` cover OpName, StaticSwitch payload, error cases. `EditorAutomationRpcGatewayTests.Build.cs` updated to add NiagaraEditor private include path for UE 5.6 header relocation.
- `#4-fix-review-issues` `IN-REVIEW` developer — Refactored `ApplyCreateNodePayload` to return `FNiagaraEditError` (dropping `FString&` out-param and colon-parse hack); added `EDITORAUTOMATIONRPCGATEWAY_API` to declaration; removed WHAT-comments from handler body; added dispatcher-path tests for `CLASS_NOT_FOUND`/`ASSET_NOT_FOUND` and `INVALID_ARGUMENT` (missing x); updated board #3 to mention `EditorAutomationRpcGatewayTests.Build.cs`.
- `#5-verify-create-node-op` `DONE` tester — Verified: `asset.duplicate` copied `/Water/Effects/Niagara/Shoreline/NiagaraShore_System` to `/Game/McpVerify/NS_McpVerifyTemp_F_niagara_graph_create_node`, then `niagara.graph.create_node` with `nodeClass: NiagaraNodeOp`, `payload.opName: Add`, `x: 240`, `y: 120`, `compile: false`, and `save: false` returned `nodeId: DF411DE648227ABC7D11C1B35C0435A6`, `nodeClass: /Script/NiagaraEditor.NiagaraNodeOp`, `compiled: false`, and `saved: false`; temp asset deleted with `asset.delete`.
