---
id: E-insert-undo-pairing-undocumented
title: "insert_*/undo_last_* pairing and 'code' vs 'BPIR' compilation distinction is undocumented (near-duplicate method descriptions)"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, blueprint, near-duplicate-verb-descriptions, undo, discovery]
encounters: 1
lastSeen: 2026-07-02T11:19:56Z
---

# `insert_*`/`undo_last_*` pairing and 'code' vs 'BPIR' compilation distinction is undocumented

The `blueprint` namespace exposes two parallel insert/undo families whose one-line
descriptions do not let a caller tell which undo reverts which insert:

- `blueprint.insert_code_at_node` — "Insert BPIR code after a selected node in the graph"
- `blueprint.insert_bpir_at_node` — "Insert BPIR code after a selected node in the graph" (**byte-identical**)
- `blueprint.undo_last_compile` — "Undo the last **code** compilation by deleting created nodes"
- `blueprint.undo_last_bpir` — "Undo the last **BPIR** compilation by deleting created nodes"

(Source-confirmed: `insert_code_at_node` + `undo_last_compile` live in
`BlueprintCodeCompilerHandler.cpp`; `insert_bpir_at_node` / `insert_bpir_before_node`
+ `undo_last_bpir` live in `BpirCompilerHandler.cpp`.) The two inserts share an
identical summary, and the two undos differ only by the words "code" vs "BPIR" —
neither of which is defined anywhere in the wiki, and nothing states which insert
pairs with which undo. The `insert_code_at_node` summary even says "Insert **BPIR**
code," further blurring the distinction from `insert_bpir_at_node`.

## Friction observed

To pick the right undo for the focus task, the agent read four near-duplicate wiki
pages (`insert_bpir_at_node`, `insert_bpir_before_node`, `undo_last_bpir` for
contrast, `undo_last_compile`) plus a Grep
(`undo_last_compile|undo_last_bpir|last compile|created nodes`) — SAY: "Now let me
understand the insert/undo pairing and BPIR syntax" — before making the correct
call. The call itself was clean once chosen (`insert_bpir_at_node` created 2 nodes;
`undo_last_compile` reported `deletedCount:2`; post-undo decompile byte-identical to
baseline). So this is pure discovery friction, not a functional bug.

## What it should do

`docs/wiki-src/blueprint.md` should:
1. Add a short "code vs BPIR compilation" note defining the two families and stating
   which `undo_last_*` reverts which `insert_*`/`compile_*` (and, if the created-node
   tracking is shared between them — the task cross-paired `insert_bpir_at_node` with
   `undo_last_compile` and it worked — say so explicitly, because that interchange is
   exactly what the near-identical descriptions leave ambiguous).
2. Give `undo_last_compile` and `insert_code_at_node` proper `### ` overlay sections
   (today only `### blueprint.insert_bpir_at_node` exists) that cross-link each insert
   to its matching undo.
3. Disambiguate the two `insert_*_at_node` summaries so they are not byte-identical.

**Docs page:** `docs/wiki-src/blueprint.md`.

## Distinct from

- `B-undo-last-bpir-doesnt-restore-phase0-sweeps`, `E-add-event-then-default-compile-bpir-unundoable`,
  `E-undo-not-reversible-suggests-nonexistent-asset-revert` — all concern the
  *behavior* of `undo_last_bpir` (Phase-0 sweeps / `UNDO_NOT_REVERSIBLE`). This ticket
  is purely the wiki **pairing/naming** gap across both families; no behavior claim.

## Evidence

From the struggle audit of a clean `blueprint.undo_last_compile` round-trip task
(namespace `blueprint`, outcome clean, 12 RPCs all `ok`, zero retries). CallAnalyzer
transcript `agent-addbb09ce332e110a.jsonl` recorded four disambiguation wiki Reads +
one Grep before the first execute. Descriptions verified in
`BlueprintCodeCompilerHandler.cpp:18-19,153-154` and `BpirCompilerHandler.cpp:359-360,717-718`.

severity rationale: impact=docs/discoverability (the methods work once the right one is chosen; only extra disambiguation reads are wasted) × reach=any caller pairing an insert with an undo (a common BPIR-authoring shape, but the docs read is a one-time cost) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean `blueprint.undo_last_compile` task (12 RPCs, all `ok`, zero retries; the focus method round-tripped perfectly). Friction: 4 near-duplicate wiki Reads (`insert_bpir_at_node`, `insert_bpir_before_node`, `undo_last_bpir`, `undo_last_compile`) + a Grep to work out which undo pairs with which insert, because `insert_code_at_node`/`insert_bpir_at_node` carry byte-identical summaries and `undo_last_compile`/`undo_last_bpir` differ only by undefined words "code" vs "BPIR". Source-confirmed descriptions in `BlueprintCodeCompilerHandler.cpp` and `BpirCompilerHandler.cpp`. Dedup: ripgrep across OPEN/DONE — the existing undo tickets (`B-undo-last-bpir-doesnt-restore-phase0-sweeps`, `E-add-event-then-default-compile-bpir-unundoable`, `E-undo-not-reversible-suggests-nonexistent-asset-revert`) are all about undo *behavior*, none about the pairing/naming docs gap. Proposed: add a code-vs-BPIR pairing note + `### ` sections for `undo_last_compile`/`insert_code_at_node` + disambiguate the two identical insert summaries in `docs/wiki-src/blueprint.md`.
