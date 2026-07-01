---
id: B-bpir-composite-inline-name-lost
title: "K2Node_Composite mid-graph emits as call K2Node_Composite() with no name or inner-graph reference"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, composite]
---

# Composite mid-graph loses name

When a `K2Node_Composite` (collapsed graph) appears in the middle of an
exec chain — i.e. as a named function-call-like block, not as the
holder of an entry event — the decompiler emits
`call K2Node_Composite() @(x, y)` and drops both:
- the composite's user-visible name
- any reference to its inner graph

Result: the BPIR text is unactionable. A reader can't tell which
collapsed graph the call refers to, and a round-trip compile cannot
reconstruct the link.

This is distinct from the already-DONE composite work
(`B-bpir-entry-points-skip-composite-subgraphs`,
`B-bpir-decompile-warnings-skip-composite-bridge`), which fixed the
case where composites *hold entry events*. This bug is about composites
used as named blocks mid-graph.

**Repro:** `App\App\UI\LobbyAndMenu\Popups\W_SaveTrack\bpir.txt` — two
composites inside an event's `Sequence` body emit as
`call K2Node_Composite() @(2368, 1605)`. 6 cached `bpir.txt` files
hit this pattern.

**Expected:** Either inline the composite body at the call site, or
emit a referencing form like `call_composite($CompositeName)` that
binds back to the composite's inner-graph definition.

**Actual:** `call K2Node_Composite() @(x, y)` with no name.

**Fix (sprint scope, decompile-only):** Add a typed branch in the BPIR decompiler dispatch for `K2Node_Composite`: extract `BoundGraph->GetName()` and emit `call K2Node_Composite(__compositeName: $Name)`. The leading-underscore arg is metadata-only — existing parser tolerates unknown args, no compile-side change. Round-trip to a re-bound composite is tracked separately in `F-bpir-composite-keyword` (full BPIR keyword surface).

## History
- `#1-initial-audit` `OPEN` reporter — 6 cached `bpir.txt` files contain `call K2Node_Composite()` mid-graph with no name. Repro on `W_SaveTrack` event body. Distinct from the DONE composite-entry-point fixes.
- `#2-decompile-only-name-preservation` `IN-REVIEW` developer — Decompile-only emit fix landed in `BpirTextEmitter.cpp` typed branch + `BpirDecompiler.cpp` dispatch wiring. Mid-graph composite now emits `call K2Node_Composite(__compositeName: $Name)`. Round-trip surface split out to `F-bpir-composite-keyword`. Test: `EditorAutomationRpcGateway.bpir.decompiler.CompositeNamePreserved`.
- `#3-superseded-by-inline` `IN-REVIEW` developer — The transitional 'call K2Node_Composite(__compositeName: $Name, ...)' decompile shape from this ticket is removed by the F-bpir-composite-keyword inline pivot. EmitCompositeNode is deleted; composites no longer appear in BPIR text. The metadata-arg shape was a stepping-stone; this ticket's fix is structurally obsoleted but not reverted.
- `#4-verified-composite-inlined` `DONE` tester — Verified: ran `blueprint.decompile` on `/App/App/UI/LobbyAndMenu/Popups/W_SaveTrack` (the original repro asset). BPIR contains zero `K2Node_Composite` and zero `composite ` keyword occurrences; the previously-broken `call K2Node_Composite() @(2368, 1605)` shape is gone, replaced by inlined sequence-body instructions in the parent flow.
