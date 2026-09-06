---
id: B-insert-bpir-rollback-leaves-anchor-exec-severed
title: "blueprint.insert_bpir_at_node splices out the anchor's existing exec edge before compiling and does not restore it when the compile fails — the rollback deletes its own nodes and leaves the rest of the function orphaned"
status: IN-REVIEW
severity: High
category: bug
tags: [blueprint, bpir, insert_bpir_at_node, rollback, atomicity, exec-splice, orphaned-nodes, silent-corruption]
encounters: 2
lastSeen: 2026-09-05T20:12:00Z
---

# A failed `insert_bpir_at_node` leaves the graph worse than it found it

`insert_bpir_at_node` on a wired exec pin splices: it breaks `anchor.<execPin> -> downstream`,
inserts its nodes between the two, then compiles the Blueprint. When that compile fails the handler
rolls back **only the nodes it created**. The severed original edge is not restored, so the anchor's
exec output is left dangling and every node downstream of it becomes unreachable.

## Repro (measured, EAContentExamples58, UE 5.8)

`/Game/FPS/Weapons/BP_WeaponBase`, function graph `TryPenetrate`, 19 nodes, `find_orphaned_nodes`
**0 orphaned** immediately before the call. The anchor `0678E13A…` is a `K2Node_IfThenElse` whose
`then` was wired to `15039AFE…` (`Line Trace By Channel`), which carries the rest of the function.

```
blueprint.insert_bpir_at_node {
  assetPath: "/Game/FPS/Weapons/BP_WeaponBase",
  nodeId:    "0678E13A4FDF4B35F9D16088064C2AA5",
  execPin:   "then",
  code:      "%pl = call Subtract_IntInt(A: $PenetrationsLeft, B: 1)\nset PenetrationsLeft = %pl"
}
->
[BLUEPRINT_COMPILE_FAILED] BPIR insert failed Blueprint compile: Variable node  Get PenetrationsLeft
 uses an invalid target.  It may depend on a node that is not connected to the execution chain, and
 got purged.
{"nodeCount":3,"createdNodes":["53E10CD5…","3A3F529B…","4493D175…"],"compiled":false,
 "status":"Error","success":false}
```

The three `createdNodes` are genuinely gone afterwards (`get_node_details_batch` on all three ->
`NODE_NOT_FOUND` x3), so the *deletion* half of the rollback works. The *reconnection* half does not:

```
blueprint.graph.get_execution_flow {graphName:"TryPenetrate"}
-> chain is Entry -> Branch -> Branch, "execOutputs": []   (3 of 19 nodes reachable)

blueprint.graph.find_orphaned_nodes {graphName:"TryPenetrate"}
-> orphanedCount: 10   (Line Trace By Channel, its Branch, the ProcessHit call, and the seven
                        data nodes feeding them)
```

Repaired by hand with one `connect_pins {0678E13A….then -> 15039AFE….execute}`, after which
`find_orphaned_nodes` returned to 0/19.

## Why this is High and not Medium

The failure mode is **silent and delayed**. The call returns an error, so a caller that treats the
error as "nothing happened" — the reasonable reading, since the created nodes really were removed —
walks away from a Blueprint whose function now stops one node into its body. Nothing downstream
complains: the orphaned half is still valid Blueprint, so a later `blueprint.compile` has no error
to report, and `asset.save`'s integrity gate sees a well-formed graph. Only
`blueprint.graph.find_orphaned_nodes` distinguishes the two states, and only if the caller happens
to run it against the *right graph* right after a call it already believes was a no-op.

The blast radius is the whole tail of the function, not the insertion point: any anchor mid-chain
takes everything after it with it. In a shared editor this lands on an asset other streams are
reading.

## What should happen

The splice and the compile should be one transaction. Either:

1. restore `anchor.<execPin> -> downstream` as part of the rollback (symmetric with the node
   deletion the handler already performs), or
2. compile-check before severing the original edge, or
3. at minimum, name the unrestored edge in the error payload
   (`"severedEdge": {"from": "<nodeId>:<pin>", "to": "<nodeId>:<pin>"}`) so the caller can repair
   it without first diffing the graph against a memory of what it used to be.

(1) is the correct fix; (3) is the cheap mitigation and would have turned this from a discovery
into a one-line repair.

## Related

- `E-compile-bpir-preexisting-errors-block-repair` — same family (a failed whole-BP compile rolls
  back placement) but about *discarded valid work*, not about the caller's pre-existing graph being
  left damaged. That ticket's rollback is lossless; this one is not.
- `B-no-undo-redo` — `blueprint.undo_last_bpir` exists but is documented as "delete created nodes",
  i.e. it is the same half-rollback and would not have restored the edge either.

severity rationale: impact=silent graph corruption that no other surface reports, on an asset the
caller believes it did not modify x reach=every `insert_bpir_at_node` into a wired exec pin whose
compile fails, which is the ordinary outcome of any BPIR snippet that is not right first time
-> High.

## Fix

`Source/PinWright/Private/Utils/BlueprintGraphSnapshot.{h,cpp}` now captures the pre-image of
every authored output-pin connection using the surviving nodes' GUIDs, pin names, and directions.
`RollbackToSnapshot` removes newly created graphs and nodes as before, then removes links absent
from the snapshot and restores every captured link that is missing, using the owning graph schema
with a direct-link fallback. This covers the anchor-to-downstream splice even when transaction
undo does not restore `LinkedTo` state. `BpirCompilerHandler.cpp` continues to use this shared
snapshot for all BPIR rollback paths; it does not keep a handler-local edge snapshot.

`Tests/Bpir/TestBpirInsertRollbackAnchorExec.cpp` now creates and saves a valid on-disk Actor
Blueprint with a CustomEvent wired to PrintString, invokes the real
`blueprint.insert_bpir_at_node` handler with `call K2Node_CallFunction()` (a node that reaches the
post-splice whole-Blueprint compile and should fail validation), and asserts the typed
`BLUEPRINT_COMPILE_FAILED` response plus symmetric restoration of the original edge. Source and
whitespace checks are complete; the Unreal compile and automation run remain verifier work.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) while adding a penetration-budget
  decrement to `BP_WeaponBase.TryPenetrate`. The BPIR snippet was wrong (the emitted
  `Get PenetrationsLeft` came out with an unresolved variable target), so the compile failing was
  correct behaviour; what was not correct is that the graph did not go back to where it started.
  Caught only because this stream runs `find_orphaned_nodes` after every graph edit against a known
  0/420 baseline — without that habit the weapon would have shipped with `TryPenetrate` doing
  nothing past its second Branch, and penetration silently dead. Worked around by reconnecting the
  edge by hand and doing the decrement with `create_node` + `connect_pins` instead.
- `#2-correction` `OPEN` reporter — **My `#1` misattributed the cause of the compile failure and the correction sharpens this ticket rather than weakening it.** The BPIR snippet was not wrong. The compile failed because an *unrelated* node elsewhere in the same graph was already broken — a `Get PenetrationsLeft` that `blueprint.graph.replace_node` had produced without self context (`B-replace-node-variableget-loses-self-context`), which `insert_bpir_at_node`'s whole-Blueprint compile then tripped over (the mechanism in `E-compile-bpir-preexisting-errors-block-repair`). Proof: after wiring that one node's `self` pin by hand, `blueprint.compile` returned `{"compiled":true,"status":"UpToDate","errors":[]}` with the BPIR nodes long since deleted. **So the severed edge is not collateral damage from bad caller input — it is what this verb does to a healthy caller whose Blueprint happens to carry a pre-existing error in any graph.** A caller who is using `insert_bpir_at_node` *to repair* a broken Blueprint — the exact case `E-compile-bpir-preexisting-errors-block-repair` is about — gets a second break for free, in a different place, every attempt.
- `#3-shared-snapshot-rollback` `IN-REVIEW` developer — Added connection pre-image capture and restoration to `Utils/BlueprintGraphSnapshot`, which is the common rollback path used by the BPIR handlers. The regression now drives the real `insert_bpir_at_node` handler on a saved valid Blueprint and supplies an invalid generic BPIR node so the handler's post-splice full compile fails; it verifies `BLUEPRINT_COMPILE_FAILED` and the original anchor-to-PrintString edge in both directions. Static source/diff checks pass; no Unreal build or automation run was performed in this implementation pass.
