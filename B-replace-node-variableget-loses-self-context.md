---
id: B-replace-node-variableget-loses-self-context
title: "blueprint.graph.replace_node produces a VariableGet with no self context — the node compiles-fails with 'uses an invalid target' and the message blames the variable, not the verb"
status: OPEN
severity: High
category: bug
tags: [blueprint, blueprint-graph, replace_node, variableget, variableset, self-context, compile-error, misleading-error]
encounters: 1
lastSeen: 2026-09-05T20:10:00Z
---

# A `replace_node` VariableGet cannot compile until you wire `self` by hand

`replace_node`'s explicit VariableGet branch creates a node whose `VariableReference` is not
self-context, so its `self` pin is a real, unconnected, required input. `create_node` on the same
variable in the same graph produces a node that needs no such wire. The compiler rejects the
`replace_node` product with:

```
Variable node  Get PenetrationsLeft  uses an invalid target.  It may depend on a node that is
not connected to the execution chain, and got purged.
```

`F-bp-graph-replace-node-rpc` (DONE) specified the opposite behaviour at design time — *"Self pin:
skip by default when replacing variable accessors … so the new node's own self context wins"* — so
this is a regression against the verb's own spec, not an unspecified corner.

## Repro (measured, EAContentExamples58, UE 5.8)

`/Game/FPS/Weapons/BP_WeaponBase`, function graph `TryPenetrate`. Swap the `Get MaxPenetrations`
feeding an `integer > integer` for a `Get PenetrationsLeft` (both plain `int` member variables on
this Blueprint, both present on the generated class — `property.get` returns
`{"value":0,"cppType":"int32","defaultSource":"class_cdo"}` for `PenetrationsLeft`):

```
blueprint.graph.replace_node {nodeId:"E4222B2A…", newNodeType:"VariableGet",
  target:"PenetrationsLeft", pinRemap:{"MaxPenetrations":"PenetrationsLeft"}}
-> {"newNodeId":"45886179…","factoryPath":"explicit","connectionsRewired":1,
    "pinRemapApplied":1,"connectionsDropped":[]}          # reports full success

blueprint.compile
-> {"compiled":false,"status":"Error",
    "errors":[{"message":"Variable node  Get PenetrationsLeft  uses an invalid target. …"}]}
```

`get_node_details` on the new node is indistinguishable from a known-good sibling — same two pins,
same types, `self` unconnected on both:

```
45886179…  Get PenetrationsLeft   pins: [PenetrationsLeft(out,int), self(in,object/BP_WeaponBase_C)]
A4BC39D5…  Get PenetrationThickness pins: [PenetrationThickness(out,real), self(in,object/BP_WeaponBase_C)]  # compiles
```

`blueprint.graph.reconstruct_node` on it does **not** fix it (`guidPreserved:true, pinCount:2`,
compile still fails). Wiring the graph's existing `K2Node_Self` into its `self` pin **does**:

```
connect_pins {C58720B0….self -> 45886179….self}
blueprint.compile -> {"compiled":true,"status":"UpToDate","errors":[]}
```

## The control that isolates the verb

In the same graph, same variable, same session, `create_node` is correct:

```
blueprint.graph.create_node {graphName:"TryPenetrate", nodeType:"VariableGet",
  variableName:"PenetrationsLeft", x:752, y:-300}  -> nodeId 2FB3B624…
```

That node compiles with its `self` pin unconnected — proven by connecting `self` on it, compiling
(still failed, on the *other* node), then breaking that link again and compiling clean. So the
variable resolves fine, `create_node` sets self context, and `replace_node` does not.

## Why the message makes this expensive

"uses an invalid target … may depend on a node that is not connected to the execution chain, and
got purged" points at a *purged dependency*, so the caller goes looking for a pruned exec branch or
a missing variable. Neither exists. Nothing in the message, and nothing in the `replace_node`
success payload, mentions self context. On this stream it cost a wrong hypothesis and a
`blueprint.insert_bpir_at_node` attempt that was itself blamed for the failure — the BPIR was fine,
it just ran a whole-Blueprint compile over an already-broken node (the mechanism
`E-compile-bpir-preexisting-errors-block-repair` describes) and its rollback then damaged the graph
further (`B-insert-bpir-rollback-leaves-anchor-exec-severed`). One defective node produced two
misattributed failures.

## What should happen

The VariableGet/VariableSet branch of `replace_node` should call `SetSelfMember` (or copy the old
node's `VariableReference` self-context flag) so the replacement is self-context, matching
`create_node` and the verb's own design note. Failing that, the response must report
`selfContext:false` so the caller can wire it, rather than returning a clean success payload for a
node that cannot compile.

Also worth checking `VariableSet` through the same branch; I only exercised `VariableGet`.

severity rationale: impact=the produced node never compiles and the compiler's message misdirects
the diagnosis entirely x reach=`VariableGet`/`VariableSet` are the first two entries in
`replace_node`'s documented vocabulary and retargeting a variable accessor is the verb's most
obvious use -> High.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) retargeting `BP_WeaponBase.TryPenetrate`'s
  penetration gate from `MaxPenetrations` to a new per-shot `PenetrationsLeft` counter. Isolated by
  the `create_node` control above; the workaround is either an explicit `K2Node_Self` wire into the
  replacement's `self` pin, or `create_node` + `connect_pins` + `delete_node` instead of
  `replace_node` for variable accessors. Took the second route in the shipped fix so the graph
  carries no cosmetic self wire.
