---
id: B-insert-bpir-rollback-leaves-anchor-exec-severed
title: "blueprint.insert_bpir_at_node splices out the anchor's existing exec edge before compiling and does not restore it when the compile fails — the rollback deletes its own nodes and leaves the rest of the function orphaned"
status: OPEN
severity: High
category: bug
tags: [blueprint, bpir, insert_bpir_at_node, rollback, atomicity, exec-splice, orphaned-nodes, silent-corruption]
encounters: 1
lastSeen: 2026-09-05T20:00:00Z
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

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) while adding a penetration-budget
  decrement to `BP_WeaponBase.TryPenetrate`. The BPIR snippet was wrong (the emitted
  `Get PenetrationsLeft` came out with an unresolved variable target), so the compile failing was
  correct behaviour; what was not correct is that the graph did not go back to where it started.
  Caught only because this stream runs `find_orphaned_nodes` after every graph edit against a known
  0/420 baseline — without that habit the weapon would have shipped with `TryPenetrate` doing
  nothing past its second Branch, and penetration silently dead. Worked around by reconnecting the
  edge by hand and doing the decrement with `create_node` + `connect_pins` instead.
