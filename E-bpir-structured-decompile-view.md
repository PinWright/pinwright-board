---
id: E-bpir-structured-decompile-view
title: "blueprint.decompile: opt-in structured code-view rendering"
status: OPEN
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
