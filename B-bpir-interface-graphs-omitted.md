---
id: B-bpir-interface-graphs-omitted
title: "BPIR decompilation omits implemented-interface graph bodies from aggregate and named output"
status: OPEN
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

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed that aggregate and named decompilation search only ubergraph, function, and macro collections at `BpirDecompiler.cpp:489-568`; implemented-interface graphs are not enumerated. No asset reproduction, build, test, editor, or MCP call was run. Severity High because successful BPIR output can silently omit executable interface bodies.
