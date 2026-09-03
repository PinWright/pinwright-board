---
id: B-bpir-dual-active-inputkey
title: "BPIR decompilation silently drops one execution branch from InputKey nodes with both Pressed and Released connected"
status: OPEN
severity: Medium
category: bug
tags: [bpir, decompiler, input-key, pressed, released, round-trip]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# BPIR drops one branch from a dual-active InputKey node

## What happens

The decompiler chooses one initial exec pin for each entry node. For an
`UK2Node_InputKey`, an active Released pin replaces that choice, and only the
resulting chain is walked
(`Source/PinWright/Private/Decompiler/BpirDecompiler.cpp:847-893`). The text
emitter likewise returns one signature per node and chooses `key_released` whenever
Released is connected (`BpirTextEmitter.cpp:1805-1824`). Entry collection records
the physical InputKey node once, not once per active exec pin
(`Handlers/Blueprint/BlueprintHandlerUtils.cpp:2391-2395,2564-2569`).

When both Pressed and Released are connected on the same node, the Released body is
selected and the Pressed body has no emitted entry. The BPIR output succeeds but is
not a complete representation of the graph.

## Why it matters

Round-tripping that output can lose input behavior silently. Severity is Medium:
silent omission is serious, but the dual-active legacy InputKey shape is narrow and
has a practical authoring workaround.

## What should happen

Model InputKey entries by node plus active exec pin. Emit independent
`key_pressed` and `key_released` entries and walk each connected body exactly once,
with regression coverage for one node carrying both branches.

## Workaround

Use separate InputKey nodes for Pressed and Released, or inspect and repair the
decompiled BPIR manually before compiling it elsewhere.

## Related

- `B-bpir-key-released-entry-silently-dropped` — wave-6 ticket fixed the
  released-only traversal and explicitly left simultaneous Pressed/Released support
  outside its scope.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed one-node entry collection, Released-over-Pressed exec selection, and one emitted signature at `BpirDecompiler.cpp:847-893`, `BpirTextEmitter.cpp:1805-1824`, and `BlueprintHandlerUtils.cpp:2391-2395,2564-2569`. No live Blueprint reproduction, build, test, editor, or MCP call was run. Severity Medium because the silent round-trip omission is limited to a dual-active legacy InputKey node and separate nodes are a workaround.
