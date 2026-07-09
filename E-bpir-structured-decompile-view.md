---
id: E-bpir-structured-decompile-view
title: "blueprint.decompile: opt-in structured code-view rendering"
status: WONTFIX
severity: Medium
category: ergonomic
tags: [bpir, decompile, readability, parity-ue58]
---

# blueprint.decompile: opt-in structured code-view rendering

BPIR decompile output is label/goto-shaped: faithful and round-trip-safe, but harder for LLMs to skim than structured code. When an agent only needs to UNDERSTAND a graph (review, explain, diff mentally), a code-shaped rendering reads far better than exec-label chains.

UE 5.8 comparison evidence: Epic's `blueprint_dsl` decompiler (`C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\EditorToolset\Content\Python\editor_toolset\toolsets\blueprint_dsl.py`, Decompiler class) reconstructs if/elif/else via join-node search, rebuilds for/while from loop macros, flips inverted empty-then branches, hoists shared pure outputs into topo-sorted binds, and strips schema-default trailing args; the output reads like a program. (Its fidelity is far worse than ours: MultiGate fails the whole decompile, Sequence flattened, comments/positions lost. The idea worth adopting is the presentation, not the pipeline.)

Proposed scope:
- `blueprint.decompile(..., view: "structured")` - opt-in READ-ONLY rendering layer: branch chains folded to if/elif/else, ForEach/While macros rendered as loops, shared pure nodes hoisted to binds, default-valued trailing pins elided.
- Canonical label output remains the default and the only compile-accepted surface; the structured view is annotated as non-compilable (or compiles by lowering back internally, developer's call, but round-trip guarantees stay on the label surface).
- Nodes that resist structuring (MultiGate, arbitrary K2 via node_props) render as today inside the structured view; never fail the decompile.

Acceptance: a graph with nested branches + a foreach decompiles to a structured view with no exec labels except where structure genuinely cannot be recovered; canonical output unchanged; all existing round-trip tests untouched.

## History
- `#1-labels-hard-to-skim` `OPEN` reporter — Label/goto decompile is round-trip-faithful but poor for LLM comprehension; Epic's blueprint_dsl code-view reconstruction (if/elif, loops, pure hoisting) is the readability target. Add opt-in structured view; labels stay canonical.
- `#2-wontfix-cosmetic-second-backend` `WONTFIX` developer — Declining (source-verified). (1) Cosmetic/Low, not a blocker: current decompile is faithful and legible via named branch(...)/foreach(...) + labeled @then:/@else:/@done: blocks (BpirDecompiler.cpp:1290-1434 EmitBranch/EmitSwitch/EmitLoop/EmitSequence; shipped legible examples bpir.examples.if-else, bpir.examples.nested-control-flow). No field is omitted, no fallback or extra calls are forced, and there is zero evidence any agent failed to comprehend the label form — the "LLMs skim structured code better" premise is unbacked; per the board Medium rubric nothing is blocked, so this is genuinely Low/pure-friction. (2) Over-scoped: no structural-recovery infrastructure exists (the decompiler only does flat label placement + a PendingLabels queue), so view:"structured" is a whole second decompiler backend — post-dominator/join recovery for if/elif/else, loop reconstruction from ForEach/While macros, pure-node hoisting via topo sort, default-arg elision, each with per-node-type fallback (MultiGate/Sequence/arbitrary K2) — not a one-iteration verified-green fix. (3) Net-negative risk: a structured view that mis-recovers control flow actively misleads the review/explain/diff use case it targets, worse than faithful verbose labels; the ticket itself concedes its reference (Epic blueprint_dsl.py) is far lower fidelity (MultiGate fails the whole decompile, Sequence flattened, comments/positions lost). (4) Premature: the reconvergence/join foundation is currently buggy — B-bpir-fallthrough-reconverge-dropped (High, IN-REVIEW, 14 encounters) shows the current decompiler already mis-renders reconvergence (shared tail folded into @then, @else dead-ends: repros #5/#8/#9/#10/#14), and a structured if/elif/else view reconstructs flow via exactly that join analysis, so it would inherit and amplify the open bug — fix the faithful primary surface first. Not a duplicate/regression/prior-decline (board-history clear); RED-TEST reproduced:false (feature request, current label output is correct). This is worth-declining on scope/risk grounds regardless of the B-bug landing, so WONTFIX rather than defer.
