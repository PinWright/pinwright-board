---
id: B-insights-export-gamethread-block
title: "`insights.export_trace` parses and exports an arbitrarily large trace synchronously on the game thread despite advertising a worker job"
status: OPEN
severity: High
category: bug
tags: [insights, trace, game-thread, unbounded-wait, documentation]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Trace export monopolizes the game thread with no job or bound

## What happens

The registration says `insights.export_trace` loads on a worker thread and returns
a job ticket (`TraceAnalysisHandler.cpp:138-150`). The implementation instead calls
`RunTraceExport` synchronously on the game thread and sends the final response
inline (`:305-316`). The core blocks in `AnalysisService->Analyze` until the whole
trace is parsed (`TraceExportCore.cpp:611-618`) and then scans providers and builds
all requested CSV contents before returning (`:631-710`). There is no timeout,
progress, cancellation or size bound.

The source comment assumes a typical trace takes about one second, but the caller
controls `tracePath`; trace size and provider work are unbounded.

## Why it matters

A large or pathological trace can freeze editor UI and starve the RPC transport for
the complete parse/export. The documented job contract also causes callers to
expect pollable progress that never exists. Severity is High for an unbounded
game-thread operation without a reproduced editor death.

## What should happen

Restore a real bounded job contract. Because the comment records that some analyzers
assert game-thread entry, do not blindly move `Analyze` to a worker. Use a bounded,
incremental game-thread state machine, an isolated analyzer process, or explicitly
limit unsupported analyzers; run UObject-free CSV work off-thread, and provide
progress, timeout and cancellation. Make the registration match the delivered
contract.

## Workaround

Use Unreal Insights externally for large traces. Limit this verb to known-small
traces while the editor is otherwise idle.

## Related

- `F-insights-export-trace` — feature ticket describes the former job design but
  does not track the current synchronous implementation.

## History
- `#1-source-scan-synchronous-export` `OPEN` reporter — Source-only scan confirmed the handler's inline game-thread call, blocking `Analyze`, whole-provider scans, and immediate final response, contradicting the registered worker/job contract. No build, test, editor, MCP call, or plugin edit was performed.
