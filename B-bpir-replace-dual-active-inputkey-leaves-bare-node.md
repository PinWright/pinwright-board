---
id: B-bpir-replace-dual-active-inputkey-leaves-bare-node
title: "A replace compile of both key_pressed and key_released for one dual-active InputKey node wipes both bodies but leaves the node itself, unwired and invisible to BPIR"
status: OPEN
severity: Medium
category: bug
tags: [bpir, compiler, replace, input-key, pressed, released, stale-node]
encounters: 1
lastSeen: 2026-10-02T00:00:00Z
rice: [1, 3, 1, 1]
priority: 33
---

# Replace keeps a bare InputKey node behind

## What happens

`CollectInputKeySubgraphNodesForEntry` (`Source/PinWright/Private/Compiler/BpirCompiler.cpp`)
returns only the entry pin's downstream subgraph when the node's *other* exec pin is also
wired, and the whole node subgraph (node included) only when it is not. A replace compile
whose text carries both `entry key_pressed K()` and `entry key_released K()` for a node with
both pins wired collects the two pin subgraphs while the other pin is still linked, so neither
pass deletes the node. After deletion the node has no exec links; `SetupKeyEvent` then authors
two new single-pin nodes for the two entries.

The leftover node is invisible in BPIR: the decompiler drops entry nodes with no connected
exec output when connected ones exist. It still binds the key at runtime (`bConsumeInput`
defaults on), so it can swallow input meant for lower-priority handlers.

## Why it matters now

`B-bpir-dual-active-inputkey` makes the decompiler emit both entries for such a node, so a
decompile -> replace-compile round trip of any Blueprint with a dual-active InputKey node now
hits this path. Source-only finding; not reproduced live.

## What should happen

When the replace wipe covers every active exec pin of an InputKey node, delete the node too
(collect both pins' entries before deciding, or delete InputKey nodes left with no exec links
after the wipe).

## Workaround

Compile into a fresh graph (`Default` mode on a Blueprint without the node), or delete the
bare InputKey node by hand after the replace.

## Related

- `B-bpir-dual-active-inputkey` - the decompiler half; its regression test round-trips through
  a fresh Blueprint in `Default` mode to avoid this path.
- `B-bpir-upsert-skips-inputkey-entries` - earlier InputKey replace identity work.

## History
- `#1-filed` `OPEN` developer - Found while fixing `B-bpir-dual-active-inputkey`. Read from source: `CollectInputKeySubgraphNodesForEntry` returns `CollectSubgraphNodesFromExecPin(<entry pin>)` whenever the opposite pin is active, so a replace naming both senses of one dual-active node deletes both bodies but never the node, and `SetupKeyEvent` adds two fresh nodes. The orphaned node is filtered out of decompiled BPIR (no connected exec) yet still binds the key. No live run, build or test was made for this ticket.
