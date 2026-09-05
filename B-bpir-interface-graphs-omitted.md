---
id: B-bpir-interface-graphs-omitted
title: "BPIR decompilation omits implemented-interface graph bodies from aggregate and named output"
status: IN-REVIEW
severity: High
category: bug
tags: [bpir, decompiler, blueprint-interface, graph-enumeration, round-trip]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# BPIR decompilation omits implemented-interface graphs

## What happens

Aggregate decompilation walks only `UbergraphPages`, `FunctionGraphs`, and
`MacroGraphs` (`Source/PinWright/Private/Decompiler/BpirDecompiler.cpp:489-532`).
Named graph lookup searches the same three collections
(`BpirDecompiler.cpp:548-568`). Neither path enumerates the graphs held by
implemented Blueprint interfaces, so those function bodies are absent from
aggregate BPIR and cannot be selected by name through this lookup.

## Why it matters

Decompilation can report success with an incomplete program, and a caller using that
text for review or round-trip can silently lose interface implementation behavior.
Severity is High under the silent-omission rule.

## What should happen

Enumerate implemented-interface graphs in both aggregate and named decompilation,
deduplicate any graph also present in a standard collection, preserve existing
AnimGraph routing, and cover aggregate plus named round-trip of an implemented
interface body.

## Workaround

Inspect each implemented-interface graph manually and preserve its body outside the
aggregate BPIR workflow.

## Related

- `B-add-function-outputs-become-inputs`
- `B-interface-function-with-outputs-unimplementable`
- `B-bpir-single-named-output-forced-returnvalue`

These are the wave-6 function-output tickets whose grouped review exposed the
decompiler enumeration gap.

## Fix

The defect was TRUE: aggregate and named graph decompilation enumerated only the
three ordinary Blueprint graph arrays, while interface implementations live under
`ImplementedInterfaces[].Graphs`. `BpirDecompiler.cpp` now builds one stable,
deduplicated list containing the ordinary arrays plus interface-owned graphs and
uses it for aggregate and named graph lookup; `decompile_function` also falls back
to the interface collection. The original graph ownership is carried through the
composite-clone path so `BpirTextEmitter.cpp` emits interface implementations as
`entry override`, which the existing `BpirParser.cpp` override branch already
accepts.

Files changed:
- `Source/PinWright/Private/Decompiler/BpirDecompiler.cpp`
- `Source/PinWright/Private/Decompiler/BpirTextEmitter.h`
- `Source/PinWright/Private/Decompiler/BpirTextEmitter.cpp`
- `Source/PinWright/Private/Tests/Bpir/TestBpirInterfaceOutputOverride.cpp`
- `Docs/wiki-src/bpir.entry-points.md`

Test added: `PinWright.bpir.decompiler.InterfaceGraphAggregateAndNamedRoundTrip`.
It invokes the compile/decompile handlers, verifies aggregate and both named read
surfaces, recompiles aggregate and graph-name output into separate targets, and
checks that the body and named output remain on the interface-owned graph without
an ordinary duplicate. Deliberately unchanged: no new BPIR entry token or compiler
semantics were added because `entry override` is already parsed and already reuses
interface-owned graphs. Per the worker brief, no build, editor, MCP, or automation
run was performed.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed that aggregate and named decompilation search only ubergraph, function, and macro collections at `BpirDecompiler.cpp:489-568`; implemented-interface graphs are not enumerated. No asset reproduction, build, test, editor, or MCP call was run. Severity High because successful BPIR output can silently omit executable interface bodies.
- `#2-interface-graphs-enumerated` `IN-REVIEW` developer — Added deduplicated interface graph enumeration for aggregate and named decompile, preserved interface ownership through cloned emission so output uses the parser-supported `entry override` form, documented the contract, and added handler-level aggregate/named round-trip coverage. No build, editor, MCP, or automation run was performed.
