---
id: E-bpir-compile-no-dangling-exec-warning
title: "compile_bpir returns empty warnings[] when a non-terminal block leaves an exec output dangling (no terminator, no fall-through) — the silent gap costs a decompile + multiple get_graph_connections probes to even notice"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [bpir, compile_bpir, warnings, lint, silent-success, ergonomics]
encounters: 4
lastSeen: 2026-06-29T04:26:10Z
---

# `compile_bpir` should warn when a block leaves an exec output dangling

`blueprint.compile_bpir` emits an **empty `warnings: []`** even when a non-terminal
block ends with an **unconnected exec output** — i.e. the last node of a block has
no `exec -> @label` terminator and no edge was created to a following node. That is
a near-certain authoring mistake (a forgotten terminator, a typo'd jump label, or
an arm the author believed would fall through), yet the response looks like a clean
`{compiled:true, status:UpToDate, errors:[], warnings:[], success:true}`. Nothing on
the `compile_bpir` response surfaces the dropped wire, so the author has to *go
looking* — decompile the graph (and reverse-engineer the inlining) or run
`get_graph_connections` and notice that one node has no outgoing exec edge — before
they can even tell something is wrong.

`bpir-gotchas` already states "`compile_bpir` is graph placement, not validation",
so this is a deliberate philosophy — but a **non-terminal block whose terminal node
has a dangling exec output** is cheaply detectable at emit time and is almost never
intentional. A single compile-time lint line, e.g.

```
warnings: ["block @else: last node's exec output is unconnected (no `exec -> @label` terminator and no fall-through edge created)"]
```

would put the problem on the very response the author is already reading, instead of
costing a multi-call diagnostic hunt.

## Evidence (from the audited round-trip-equivalence task on `blueprint.compile_bpir`)

Authoring the documented §2.8 / `bpir.examples.if-else` reconvergence idiom (true arm
`@then ... exec -> @done`; false arm `@else ...` with no terminator, expected to fall
through to the shared `@done` tail) into a fresh Actor BP, `compile_bpir` returned
`{nodeCount:8, createdNodes:[...], warnings:[]}` — full success, **zero warnings** —
even though the `@else` block's terminal `PrintString` (node `412F8E05…`) was left with
a **dangling, unconnected exec output**. `get_graph_connections (edgeType:exec)` later
confirmed that node has **no outgoing exec edge** and the shared tail has a single exec
predecessor. Because the compile reported a clean success, the author could only
discover the dropped wire by spending **~6 recovery calls**: a `blueprint.decompile`
(ambiguous — the tail gets inlined into `@then`), a `get_graph_connections`, an
unpositioned diagnostic recompile, a second `get_graph_connections`, a second
`create + compile_bpir` with explicit `exec -> @done` on both arms, and a final
`get_graph_connections` confirm. A warning on the **first** `compile_bpir` response
would have collapsed that hunt to one call.

## Relationship to other tickets

- This is **complementary to** `B-bpir-fallthrough-reconverge-dropped` (the High bug:
  the fall-through reconvergence edge is silently dropped when the tail is already
  wired by a sibling's explicit jump). That ticket fixes the **wiring**; this ticket
  asks for a **diagnostic** that survives the fix: even once fall-through works (or once
  the docs require explicit terminators), *asymmetric* branch authoring — a genuinely
  forgotten terminator, a mis-typed `@label`, an arm the author meant to dead-end vs one
  they didn't — will keep producing a dangling exec output with no signal. The bug
  ticket's own fix sketch proposes "at minimum surface `COMPILE_FAILED`" only for the
  specific already-wired-tail case; this lint generalizes that to *any* non-terminal
  block with an unconnected exec output, at warning (not failure) severity so legitimate
  intentional dead-ends are not blocked.
- Not the same as `B-false-compile-success` / `E-bpir-auto-validate` (both DONE): those
  added UE-`blueprint_compile` errors/warnings to the response. A dangling exec output is
  **not** a UE compile error (UE considers an unexecuted node valid), so it slips past
  that surface entirely — the lint has to be a BPIR-level emit-time check, not a
  pass-through of the Kismet compiler's diagnostics.

**Fix:** after exec wiring, for each block whose terminal node-backed statement carries
an exec-output pin that ended up with zero links and which has no authored
`exec -> @label` terminator, append a `warnings[]` entry naming the block/label and the
node. Keep it a warning (not an error) so deliberate dead-ends remain authorable.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a `blueprint.compile_bpir` round-trip-equivalence task. `compile_bpir` of the documented §2.8 reconvergence idiom returned `{nodeCount:8, warnings:[]}` — clean success, zero warnings — while the `@else` block's terminal `PrintString` (`412F8E05…`) was left with a dangling, unconnected exec output (`get_graph_connections` confirmed no outgoing exec edge; shared tail had a single exec predecessor). The silent success cost the author ~6 recovery calls (decompile + get_graph_connections + unpositioned recompile + get_graph_connections + a second create/compile with explicit `exec -> @done` on both arms + a final get_graph_connections) to discover the dropped wire. Proposing a cheap emit-time lint that names the block whose terminal node has an unconnected exec output, so the dropped wire shows on the first compile response instead of a multi-call hunt. Complementary to (not a dup of) `B-bpir-fallthrough-reconverge-dropped` (wiring fix) — this is the missing diagnostic that outlives that fix for any asymmetric/typo'd branch authoring. Distinct from DONE `B-false-compile-success` / `E-bpir-auto-validate` (UE-compile pass-through: a dangling exec is not a UE compile error). Surfaced by the CallAnalyzer trace + friction note on the audited task; the round-trip readback was faithful — the only divergence was the silently dropped exec edge.
- `#2-cross-task-roundtrip-probe-reconfirm` `OPEN` reporter — Cross-task reconfirm (struggle-audit of another `blueprint.compile_bpir` round-trip-equivalence task, fresh Actor BP `/Game/BP_RoundTripProbe`). `compile_bpir` of the §2.8 idiom (true-arm `exec -> @merge`; false-arm fall-through to the shared `@merge`) returned `{nodeCount:7, createdNodes:[...], errors:[], warnings:[], compiled:true, status:"UpToDate", success:true}` — clean success, **zero warnings** — while the false-arm `PrintString` was left with a dangling, unconnected exec output (`get_graph_connections (edgeType:exec)`: 4 exec edges, false-arm node no outgoing exec, `@merge` single predecessor). The silent success cost ~4 extra recovery calls (`blueprint.decompile` [ambiguous — merge inlined under `@then`] + `get_graph_connections` + `find_nodes` + a control recompile with explicit `exec -> @merge` on BOTH arms + a confirming `get_graph_connections` showing 5 edges / two predecessors). A dangling-exec `warnings[]` line on the first `compile_bpir` response would have collapsed the hunt to one call. Same friction as `#1`; complementary to the wiring bug `B-bpir-fallthrough-reconverge-dropped` (this audit's `filed_id`). Severity unchanged Medium.
- `#3-cross-task-manual-placement-reconfirm` `OPEN` reporter — Third cross-task reconfirm (struggle-audit, transcript `agent-a1cc1fa9088bfccf0.jsonl`, 19 RPCs all `ok`/non-error, focus `blueprint.compile_bpir`, fresh Actor BP `/Game/BP_BpirRoundTrip`). Authored a manual-placement `custom_event` whose true arm `@big` does explicit `exec -> @done` and whose false arm `@small` (terminal `call PrintString(InString:"small path") @(4937,113)`, no terminator) falls through to the shared `@done`. `compile_bpir` returned `{nodeCount:6, createdNodes:[...6...], errors:[], warnings:[], compiled:true, status:"UpToDate"}` — clean success, **zero warnings** — while `get_graph_connections (edgeType:exec)` returned only **4** exec edges (`@small`'s "small path" `PrintString` has no edge to `@done`'s "done" `PrintString`; `@done` single predecessor). The silent success cost ~5 extra recovery calls vs the planned single `compile_bpir`: a re-compile adding explicit `exec -> @done` to `@small` → 5 exec edges (correct), a re-compile of an UNPOSITIONED fall-through variant → again 4 edges (proving the drop is not manual-placement-specific), 3 `get_graph_connections`, and a `blueprint.decompile` (ambiguous — `done` inlined into `@then`). One dangling-exec `warnings[]` line on the first response would have surfaced it instantly. Same friction as `#1`/`#2`; complementary to the wiring bug `B-bpir-fallthrough-reconverge-dropped`. Severity unchanged Medium.
- `#4-cross-task-control-bp-reconfirm` `OPEN` reporter — Fourth cross-task reconfirm (struggle-audit, transcript `agent-a41684d4aee4673b2.jsonl`, 19 RPCs all `ok`/non-error, focus `blueprint.compile_bpir`). Fresh Actor BP `/Game/BP_BpirRoundTrip` authored the documented §2.8 / `bpir.examples.if-else` reconvergence idiom (true arm `exec -> @done`; false arm `@else` falls through to the shared `@done`); `compile_bpir` returned `{nodeCount:9, errors:[], warnings:[], compiled:true, status:"UpToDate", success:true}` — clean success, **zero warnings** — while `get_graph_connections (edgeType:exec)` showed only **5** exec edges, the `@else` "False path" node (`53AAB8AC…`) with **no** outgoing exec edge, and the shared tail "Reconverged" with a **single** exec predecessor. Because the compile reported clean, the agent could only discover the dropped wire by standing up a **whole second control Blueprint** (`/Game/BP_BpirReconvCtl`) plus ~5 extra probes: an unpositioned doc-shape if-else recompile (still dead-ends), a reduced **trivial two-label sequential `custom_event`** (`call A` / `@next:` / `call B` → `connectionCount:1`, A never reaches B — matching the bug ticket's `#12` plain-sequential-fall-through scope), and several `get_graph_connections` — before re-authoring with explicit `exec -> @done` on both arms. A single dangling-exec `warnings[]` line on the FIRST `compile_bpir` response would have collapsed that second-BP forensic hunt to one call. Same friction as `#1`–`#3`; complementary to the wiring bug `B-bpir-fallthrough-reconverge-dropped` (this audit's `filed_id`). Severity unchanged Medium.
- `#5-already-resolved-by-fallthrough-wiring` `IN-REVIEW` developer — ALREADY-FIXED in current synced source; no code added, not compiled. All four encounters (#1–#4) reproduce the SAME scenario — the documented §2.8 / `bpir.examples.if-else` reconvergence idiom (true arm `exec -> @done`/`@merge`; false arm `@else` with no terminator, expected to fall through to the shared tail) — where the false arm's terminal `PrintString` was left with a dangling exec output. That dangling output is the SYMPTOM of the wiring bug `B-bpir-fallthrough-reconverge-dropped`, whose fix (#13, IN-REVIEW) is already present here: the `FallThroughExec` cross-segment carry in `WireExecPins` Step 2 — `Source/PinWright/Private/Compiler/BpirCompiler.cpp:7063` (seed `FallThroughExec = EntryExecPin`) and `:7147` (`FallThroughExec = LastExecOutput`), header comment naming this bug at :7048-7059 — now feeds each open-ended block's trailing exec output into the next adjacent label, so the §2.8 idiom no longer dangles. Locked by regression test `PinWright.bpir.compiler.integration.FallThroughReconvergence` (`Source/PinWright/Private/Tests/Bpir/TestCompilerIntegration.cpp:4788`), which asserts the `@else` output is wired specifically into the shared tail AND the tail has exactly 2 exec predecessors. Re-running any of the four repros in current source produces NO dangling output — the ticket's documented defect no longer matches reality. The requested `warnings[]` lint was NOT implemented and has no evidenced residual target: a mis-typed/missing `exec -> @label` is already a LOUD compile error (`BpirCompiler.cpp:7404-7407` "ExecGoto target label '@%s' not found"; post-wire verification at :7315-7319 raises a COMPILE_FAILED for a resolved-but-unwired labeled target), a forgotten terminator now auto-falls-through, and the only remaining genuine dangles (an intentional mid-graph dead-end, or the normal final block of a chain) are legitimate — a warning there is a false positive that contradicts the documented "`compile_bpir` is graph placement, not validation" philosophy. Tester: verify the §2.8 fall-through wires (run `FallThroughReconvergence`, or compile a fresh §2.8 idiom and confirm the shared tail has 2 exec predecessors with no dangling `@else` output) and close DONE.
