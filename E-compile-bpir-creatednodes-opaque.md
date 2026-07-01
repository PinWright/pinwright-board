---
id: E-compile-bpir-creatednodes-opaque
title: "blueprint.compile_bpir returns a bare createdNodes GUID list with no type/role/position, so an injected helper node can't be detected from the compile result — forcing a follow-up get_nodes"
status: OPEN
severity: Low
category: ergonomic
tags: [bpir, compile_bpir, response-shape, createdNodes, verification, ergonomics]
encounters: 2
lastSeen: 2026-06-28T23:00:01Z
---

# `blueprint.compile_bpir` createdNodes is an opaque GUID list — callers can't tell authored from synthesized nodes without a follow-up call

`blueprint.compile_bpir` reports what it built as a bare list of node GUIDs plus
a count: `{"nodeCount":6, "createdNodes":["88606E84…","FF22A7D3…",…],
"compiled":true, "status":"UpToDate", "success":true}`. The list carries **no
per-node type, no authored-vs-synthesized role flag, and no `x`/`y`** — so when
the compiler injects an implicit/visible helper (a `K2Node_Self`, a
`BreakHitResult`/`BreakVector`, a `VariableGet`, etc.), the caller cannot see it
in the compile result. The only tell is that `nodeCount` exceeds the number of
visible instructions the caller authored, and even then the surplus GUID is
anonymous: to learn *which* node it is, what type it is, and where it landed, the
caller must make a separate `blueprint.graph.get_nodes` round-trip.

## What it should do

Label each entry of `createdNodes` in the compile response with the cheap
metadata the compiler already has on hand when it creates the node:
`nodeType` (e.g. `K2Node_Self`, `K2Node_CallFunction`), an
`authored` | `synthesized` role flag (was this node backed by a BPIR
instruction line, or auto-inserted by data-pin wiring / layout?), and its
`x`/`y`. With that, a caller can detect an injected/implicit helper — and
whether its position is authorable — straight from the compile result, with no
follow-up inspection call. (This is response enrichment on `compile_bpir`'s
existing output, not a new verb.)

## Evidence

From the struggle audit of a clean-process BPIR authored-position FORMATTING
round-trip probe (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome
`tool_bug` filed as `B-bpir-positioned-implicit-helper-not-rejected`; 6 RPCs, all
`ok`/non-error, zero retries) on a fresh Actor BP `/Game/BP_BpirPositionRoundTrip`
(BeginPlay → `IsValid(self)` → branch → 2× PrintString, every node `@(x,y)`).
`compile_bpir` returned `{"nodeCount":6, "createdNodes":["88606E84…","FF22A7D3…",
"415A775A…","D74E0183…","BD7489DC…","59D4CA2C…"], "compiled":true, "success":true}`
for only **5** authored visible instructions. The agent's THINK, verbatim:
*"reports nodeCount:6 — I authored only 5 visible nodes … A 6th node was created.
Let me inspect all nodes."* It then had to call `blueprint.graph.get_nodes`
purely to identify the surplus GUID `FF22A7D3…` as an auto-injected `K2Node_Self`
helper at `(-416,13416)` — a call that itself overflowed the inline budget and
spilled to a `HttpResponses/<uuid>.json` file the agent had to `Read`. Had
`createdNodes` carried `{nodeType, authored|synthesized, x, y}`, the injected
helper would have been visible in the compile result with no extra round-trip
(and no spill). Recurs cross-task on a milder surface — see `#2`.

## Distinct from

- `B-bpir-positioned-implicit-helper-not-rejected` (OPEN, judge-filed for this
  same task) — the **correctness** bug: the authored-position guard that should
  `COMPILE_FAILED` on an implicit visible helper is not enforced, so the helper
  is silently injected. This ticket is the orthogonal **ergonomic** angle: even
  setting the guard aside (in normal/auto-layout mode implicit helpers are
  *legitimate*), the compile response gives no way to see what was created, so
  the surplus node is invisible without a second call. Fixing the guard removes
  the helper in *positioned* mode only; this enrichment helps every mode.
- `E-get-nodes-pins-spill-no-projection` (OPEN) — that ticket fixes the *spill*
  on the follow-up `get_nodes` (add a `fields`/`nodeIds` projection). This ticket
  removes the *need* for that follow-up at all by labeling the nodes at their
  point of creation; complementary, different method/surface.
- `F-bp-graph-integrity-snapshot` (OPEN) — proposes a **new** bundled read RPC
  (`blueprint.graph.snapshot` = decompile + connections + orphans). This ticket
  enriches `compile_bpir`'s **existing** response in place; no new verb.
- `E-compile-bpir-idempotent-omits-guid-regen` (OPEN) — docs-only, about GUIDs
  regenerating across re-applies. Orthogonal: that is about GUID *stability*,
  this is about the GUID list being *uninformative* on a single apply.
- `B-bpir-positioned-implicit-helper-not-rejected` (OPEN, High) — its `#2`/`#4`
  cover this same `IsValid(Object: self)` → `K2Node_Self` trigger as a
  **correctness** bug (the helper should be rejected / never spawned). A
  CallAnalyzer pass on the `#2`-below task proposed the *opposite* docs framing
  (document the Self node as an *allowed transparent* helper that counts toward
  `nodeCount`); that "allowed vs rejected" decision is owned by the bug's
  resolution and is deliberately **not** split into a separate docs ticket here.
  This ergonomic ticket is mode-agnostic: whatever the verdict, the compile
  result should still label which `createdNodes` are authored vs synthesized.

Severity Low: pure friction — a response that omits cheap metadata and forces one
follow-up inspection call. `compile_bpir` runs in nearly every BPIR task, but the
gap only bites when a caller wants to verify the created-node set (not every
call), so it stays Low rather than being bumped for reach.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the CallAnalyzer trace of a clean-process BPIR authored-position round-trip probe (focus `blueprint.compile_bpir`, `/Game/BP_BpirPositionRoundTrip`, 6 RPCs all `ok`/non-error, zero retries; outcome `tool_bug` → `B-bpir-positioned-implicit-helper-not-rejected`). `compile_bpir` returned `nodeCount:6` + 6 bare GUIDs for **5** authored visible instructions; the agent's THINK (*"A 6th node was created. Let me inspect all nodes."*) shows it had to issue a separate `blueprint.graph.get_nodes` (which itself overflowed and spilled to a `HttpResponses` file) purely to identify the surplus GUID as an auto-injected `K2Node_Self` at `(-416,13416)`. The response carries no `nodeType`/role/`x`/`y` per created node, so injected helpers are undetectable from the compile result alone. Proposed: label each `createdNodes` entry with `{nodeType, authored|synthesized, x, y}` so callers see injected/implicit nodes without a follow-up call. Dedup: ripgrep across OPEN/closed — distinct from the correctness bug `B-bpir-positioned-implicit-helper-not-rejected` (guard not enforced), from `E-get-nodes-pins-spill-no-projection` (fixes the follow-up's spill, not the need for it), from `F-bp-graph-integrity-snapshot` (new bundled read verb), and from `E-compile-bpir-idempotent-omits-guid-regen` (GUID-regen docs). No ticket proposes enriching `compile_bpir`'s `createdNodes` with type/role/position.
- `#2-cross-task-self-node-nodecount-milder` `OPEN` reporter — Cross-task
  recurrence on a milder surface (struggle audit of a BPIR authored-position
  FORMATTING round-trip probe, focus `blueprint.compile_bpir`, namespace
  `blueprint`, fresh Actor BP `/Game/BP_BpirRoundTrip`; 6 RPCs, all `ok`/non-error,
  zero retries; outcome `clean`/no judge filing). `compile_bpir` returned
  `{nodeCount:8, createdNodes:[8 GUIDs], errors:[], warnings:[], compiled:true,
  success:true}` for only **7** authored node-backed instructions (BeginPlay:
  entry/`IsValid`/branch/2× PrintString + Tick: entry/PrintString). Same root gap
  as `#1`: the surplus 8th GUID is an anonymous implicit `K2Node_Self` for
  `Object: self` in the authored-position BeginPlay, and the bare GUID list gives
  no `nodeType`/role/`x`/`y` to identify it. Agent friction note, verbatim:
  *"compile reported 8 nodes vs the 7 BPIR lines I authored, which briefly looked
  like a phantom-helper risk in authored-position mode -- it was an implicit
  UK2Node_Self for `Object: self`, but the decompiler correctly treats it as
  transparent (inline `self`)."* THINK confirms it had to register the doubt and
  fall through to `blueprint.decompile` to resolve it (decompile returned
  `%n0: bool = call IsValid(Object: self) @(336, 32)` with every authored `@(x,y)`
  verbatim and NO separate Self line — folded back to `self`). **Milder than `#1`:
  the round-trip task already required a decompile, so verification cost *no extra
  RPC* and triggered no `get_nodes` spill — but the created-node set was still
  unverifiable from the compile result alone, so `nodeCount` is a misleading
  "no phantom nodes" signal that forced a doubt-then-confirm step. Reinforces the
  enrichment ask: label each `createdNodes` entry `{nodeType, authored|synthesized,
  x, y}` (or split `primaryNodeCount` vs `helperNodeCount`) so the
  success-check clause "the body contains exactly the authored primaries, no extra
  helper/layout nodes" is directly checkable from the compile result.** Dedup: the
  CallAnalyzer's docs sub-angle (document the Self node as an allowed transparent
  helper) belongs to / conflicts with the correctness bug
  `B-bpir-positioned-implicit-helper-not-rejected` (`#2`/`#4`, same trigger), not
  this ergonomic ticket — see "Distinct from" above; no separate docs ticket filed.
