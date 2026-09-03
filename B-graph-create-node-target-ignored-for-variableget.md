---
id: B-graph-create-node-target-ignored-for-variableget
title: "blueprint.graph.create_node ignores the documented `target` slot for VariableGet/VariableSet and fails with VARIABLE_NOT_FOUND: Variable '' — only the legacy `variableName` alias works"
status: OPEN
severity: Medium
category: bug
tags: [blueprint, blueprint-graph, create_node, variableget, variableset, target, alias, docs-mismatch]
encounters: 1
lastSeen: 2026-09-02T19:47:00Z
---

# `target` is documented for `VariableGet` and silently unused

## Repro

```
blueprint.graph.create_node {assetPath:"/Game/FPS/UI/Test/BP_HUDTestPawn",
  graphName:"GetHealth", nodeType:"VariableGet", target:"Health01", x:16, y:128}
-> [VARIABLE_NOT_FOUND] Variable '' not found
```

Five calls of that shape (variables `Health01`, `Spread`, `WeaponNameVar`, `MagAmmo`,
`ReserveAmmo`, all existing and all listed by `blueprint.inspect`) failed identically. The empty
quotes in `Variable '' not found` are the tell: the handler read its variable-name slot, found
nothing there, and never looked at `target`.

Swapping one field, everything else identical, works:

```
blueprint.graph.create_node {assetPath:"...", graphName:"GetHealth",
  nodeType:"VariableGet", variableName:"Health01", x:16, y:128}
-> {"nodeId":"DC5994A94A275BED4A2E91BD4ACDBE5E","nodeName":"K2Node_VariableGet_0"}
```

## The wiki says `target` should work

`Saved/PinWright/wiki/blueprint.graph.create_node.md` lists `VariableGet` / `K2Node_VariableGet`
and `VariableSet` / `K2Node_VariableSet` first under **"Supported target-aware node types"**, gives
the rule under **"`target` interpretation by type"** — *"CallFunction / VariableGet / VariableSet /
Event — function, variable, or event name"* — and even carries a worked example:

```json
{ "assetPath": "/Game/BP_Foo.BP_Foo", "nodeType": "VariableGet", "target": "CurrentLap", "x": 300, "y": 200 }
```

That example does not work. The page then describes `variableName` as a **legacy alias** accepted
"when `target` is absent", which inverts the actual behaviour: `variableName` is the only field
that works and `target` is the one that is ignored.

## What should happen

`target` should resolve the variable for `VariableGet` / `VariableSet`, matching its own
documentation and the behaviour of the other target-aware types. Failing that, the page must be
corrected and the handler must reject an unusable `target` explicitly rather than reporting a
missing name it was never given — `VARIABLE_NOT_FOUND: Variable ''` sends the caller looking for a
missing variable instead of a misread parameter.

Worth checking whether `CallFunction` and `Event` have the same gap; I only exercised `VariableGet`.

**Workaround:** use `variableName` for variable nodes.

severity rationale: impact=soft blocker — the documented parameter is inert and the error
misdirects, but the legacy alias works and is on the same page x reach=`create_node` is the
surgical-edit path used whenever `compile_bpir` cannot reach a graph (e.g. Blueprint Interface
implementation graphs, which is exactly where I needed it) -> Medium.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) while implementing `BPI_HUDSource` on `/Game/FPS/UI/Test/BP_HUDTestPawn`. `blueprint.compile_bpir` cannot author a Blueprint Interface implementation graph at all (`Found more than one function with the same name` when the interface graphs exist; `INTERFACE_MUTATION_FAILED` if you create the functions first — see `B-interface-function-with-outputs-unimplementable`), so `blueprint.graph.create_node` + `connect_pins` into the existing interface graph was the only route left. Five `target:"<VarName>"` calls returned `VARIABLE_NOT_FOUND: Variable ''`; the same five with `variableName:"<VarName>"` all returned node ids, and the resulting graphs compile and decompile correctly. The wiki page lists `VariableGet` under "Supported target-aware node types" with a worked `target` example, and calls `variableName` a legacy alias — the opposite of what the handler does.
