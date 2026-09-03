---
id: E-bpir-replacement-root-scan
title: "Replace-mode BPIR rescans every Blueprint graph and node once for each authored block"
status: OPEN
severity: Low
category: ergonomic
tags: [bpir, compiler, replace-mode, performance, graph-scan]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# Replace-mode BPIR repeatedly scans every graph and node

## What happens

`FindEntryNodeForBlock()` builds a graph list from the current graph plus every
ubergraph, function graph, and macro graph, then scans every node until it finds a
match (`Source/PinWright/Private/Compiler/BpirCompiler.cpp:2339-2366`). Compile
calls it for every parsed block (`BpirCompiler.cpp:3376-3379`). Large replacement
documents therefore do work proportional to authored blocks multiplied by graph
nodes, with the current graph potentially present twice in the list.

## Why it matters

Large BPIR replacements become progressively slower even though the Blueprint's
entry set is stable during this phase. Severity is Low under the board rubric: this
is performance friction on unusually large inputs, not incorrect output or a hard
blocker.

## What should happen

Build an entry lookup once per compile phase, or retain the authored entry node when
`SetupEntryPoint()` creates or resolves it. Avoid a full Blueprint scan per block and
cover a multi-block replacement with a structural or bounded-work regression test.

## Workaround

Split a large replacement document into smaller calls, accepting the extra compile
and coordination overhead.

## Related

- `F-author-enhanced-input-action-node` — wave-6 BPIR authoring ticket whose review
  exposed this repeated lookup.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed the all-graph/all-node scan at `BpirCompiler.cpp:2339-2366` and its per-block call at `:3376-3379`. No benchmark, build, test, editor, or MCP call was run. Severity Low because the effect is scale-dependent latency with a document-splitting workaround.
