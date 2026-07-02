---
id: E-add-event-then-default-compile-bpir-unundoable
title: "add_event then default compile_bpir is silently un-undoable; error points backward not to mode=extend"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [bpir, undo, docs, compile_bpir]
---

# add_event then default compile_bpir is silently un-undoable; error points backward not to mode=extend

The most natural "prototype then back out" workflow for a custom event is a trap with
no forward-pointing guidance:

1. `blueprint.add_event` (CustomEvent) — creates an entry node.
2. `blueprint.compile_bpir` with the event body in **default** mode (`mode: "append"`,
   the documented default) — this runs a Phase 0 sweep that **deletes the
   `add_event`-created entry** (not snapshotted for restore) before re-creating it.
3. `blueprint.undo_last_bpir` — fails with `UNDO_NOT_REVERSIBLE`, because the compile
   it would reverse ran a Phase 0 sweep.

So the obvious prototype-and-revert sequence is un-undoable, and nothing up front warns
the caller. This is distinct from the two existing tickets:
- `B-undo-last-bpir-doesnt-restore-phase0-sweeps` (DONE) fixed the *behavior* — undo now
  correctly *refuses* rather than half-rolling-back. Correct, but it doesn't make the
  workflow undoable.
- `E-undo-not-reversible-suggests-nonexistent-asset-revert` (OPEN) is narrowly about the
  dangling `asset.revert` string in the error.

The PROCESS gap neither covers: there **is** a fully-undoable path —
`compile_bpir { mode: "extend" }` skips the Phase 0 sweep, so a later `undo_last_bpir`
cleanly deletes exactly the nodes it created (verified this task: extend-mode compile then
undo returned `deletedCount: 5` and the decompile was byte-identical to the baseline). But:
- The `UNDO_NOT_REVERSIBLE` error points **backward** (`git revert` / `asset.revert`) when
  for an *additive* prototype the right remedy is **forward**: "this compile ran in
  append/replace mode; to keep undo working, recompile the body with `mode: "extend"`."
- The `blueprint.bpir-gotchas.md` wiki page (which the caller navigated this task) has a
  bullet on default mode mapping to the Phase-0-deleting upsert path, but frames it only as
  "retries are idempotent" and says to use `extend` "only when intentionally appending to an
  existing override body." It never connects this to undoability, nor warns that
  `add_event` + default `compile_bpir` is un-undoable.

**What it should do (any of):** (a) `blueprint.bpir-gotchas.md` gains a bullet: "If you may
want to `undo_last_bpir` a body you're adding to a fresh `add_event` entry, compile with
`mode: "extend"` — default `append` mode runs a Phase 0 sweep that makes the compile
un-undoable." (b) The `UNDO_NOT_REVERSIBLE` error gains a forward hint pointing at
`mode: "extend"` for additive cases, not only git. (c) Optionally, `undo_last_bpir` could
also un-sweep when the only swept entry was created by the immediately-preceding `add_event`
in the same session (out of scope here; a feature, not this docs ergonomic).

**Docs page to improve:** `docs/wiki-src/blueprint.bpir-gotchas.md`.

**Evidence (this task — focus `blueprint.undo_last_bpir`, BP_Light_Bulb_Basic):** friction note
verbatim — *"the first undo_last_bpir failed with UNDO_NOT_REVERSIBLE because the default
append-mode compile_bpir in step4 ran a Phase 0 sweep deleting the pre-existing
add_event-created FlickerOnce entry (not snapshotted for restore); I recovered by deleting the
leftover nodes via blueprint.graph.delete_node, re-compiling with mode=extend (skips the
sweep), then undo cleanly deleted its 5 nodes — a real ergonomic gap, since the prescribed
add_event-then-default-compile sequence is silently un-undoable."* Recovery cost: 4 extra
calls after the failed undo (`graph.find_nodes`, `graph.delete_node`, `decompile`,
`compile_bpir mode=extend`) plus the retried `undo_last_bpir` — all to discover a forward path
the error and wiki never named.

## History
- `#1-initial-audit` `OPEN` reporter — Process angle distinct from `B-undo-last-bpir-doesnt-restore-phase0-sweeps` (DONE, behavior) and `E-undo-not-reversible-suggests-nonexistent-asset-revert` (OPEN, dangling `asset.revert` string): the `add_event` → default `compile_bpir` → `undo_last_bpir` sequence is silently un-undoable, and neither the error nor `blueprint.bpir-gotchas.md` points the caller at the undoable forward path (`mode: "extend"`). Replayed on `/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic`; first `undo_last_bpir` returned `UNDO_NOT_REVERSIBLE`; recovery required delete-leftover-nodes + recompile with `mode=extend` (then undo cleanly returned `deletedCount: 5`). 4 extra recovery calls + 1 retried undo.
- `#2-docs-fix` `IN-REVIEW` developer — Implemented the ticket's primary remedy (option a): added a bullet to `Docs/wiki-src/blueprint.bpir-gotchas.md` spelling out that `add_event` + default `append`/`replace` `compile_bpir` runs a Phase 0 sweep that makes the compile un-undoable, that `undo_last_bpir` then correctly refuses with `UNDO_NOT_REVERSIBLE`, and that the cheap *forward* remedy is to compile the body with `mode: "extend"` (skips Phase 0, so undo cleanly deletes exactly the created nodes). Deliberately did NOT touch the `UNDO_NOT_REVERSIBLE` SendError at `BpirCompilerHandler.cpp:747-751` (option b) to avoid colliding with the OPEN sibling `E-undo-not-reversible-suggests-nonexistent-asset-revert` that owns that string; option c (auto-un-sweep) is out of scope. Regression test: added `FBpirUndoLastBpirSucceedsAfterExtendTest` (`undo_last_bpir.SucceedsAfterExtend`) to `Source/EditorAutomationRpcGateway/Private/Tests/Bpir/TestBpirUndoLastBpir.cpp` — drives the production handler end-to-end (default seed compile → `mode:"extend"` append → `undo_last_bpir`) and asserts the extend compile pushed `false` on `Phase0RanStack` (Phase 0 skipped) and that undo SUCCEEDS, which fails if the `bPhase0Ran = (Mode == Replace)` contract that the docs guidance relies on is ever reverted. Files: `Docs/wiki-src/blueprint.bpir-gotchas.md`, `Source/EditorAutomationRpcGateway/Private/Tests/Bpir/TestBpirUndoLastBpir.cpp`.
- `#3-cross-task-evidence` `IN-REVIEW` reporter — Cross-task corroboration (struggle audit, focus `blueprint.compile_bpir`, `/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic`): a default-mode `compile_bpir` need NOT be preceded by `add_event` to be un-undoable. Here a *fresh* all-`@(x,y)` event compile (`FuzzImplicitHelper`, wrongly accepted per `B-bpir-positioned-implicit-helper-not-rejected`) returned `success:true` but left 4 orphan helper nodes (`find_orphaned_nodes` → `orphanedCount:4`: Self, Get Actor Location, Conv Double to Float, Break Vector); the follow-up `undo_last_bpir` again returned `[UNDO_NOT_REVERSIBLE]` (append/replace Phase 0 sweep). Notably the documented forward remedy (`mode:"extend"`) would NOT have helped this case — the dirt was orphaned helper nodes, not a clean append — so recovery took a *different* 3-call teardown: `blueprint.remove_event` (removedNodeCount:2) + `blueprint.graph.delete_orphaned_nodes` (deletedCount:4). Reinforces that ANY default `compile_bpir` is un-undoable, and that a caller leaning on `undo_last_bpir` for rollback hits the same wall regardless of how the graph got dirty (here a buggy accept, not an intentional prototype). Same root friction; no new ticket — logged here for recurrence.
