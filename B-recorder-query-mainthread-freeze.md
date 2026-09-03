---
id: B-recorder-query-mainthread-freeze
title: "`recorder.query` executes unrestricted caller Python synchronously on the editor main thread with no timeout or cancellation"
status: OPEN
severity: High
category: bug
tags: [recorder, python, game-thread, unbounded-wait, editor-freeze]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# An unbounded recorder query can freeze the editor indefinitely

## What happens

`recorder.query` embeds the caller's complete function body in a temporary Python
file (`RecorderQueryHandler.cpp:71-104`) and runs it through synchronous
`ExecPythonCommandEx` at `:239`. The handler has no job, deadline, watchdog or
cancel path. The registration admits that the snippet runs on the editor main
thread and that an unbounded loop freezes it (`:139-155`).

A valid request whose body is `while True: pass` never returns to the handler, the
RPC transport, or normal editor ticking. The `maxRows` cap only applies after the
caller code returns and does not bound execution.

## Why it matters

One caller-controlled request can wedge the shared editor and all other RPC work
without a recovery path. Severity is High for an exposed unbounded main-thread
operation without a reproduced editor death.

## What should happen

Do not execute arbitrary caller loops on the game thread. Run the query in an
isolated process or another safely terminable boundary, expose it as a cancellable
job, and enforce a wall-clock and resource budget. A worker-thread move alone is
insufficient if the embedded API can touch Unreal objects; otherwise restrict the
query language to bounded data operations.

## Workaround

Use the structured recorder verbs. If `recorder.query` is unavoidable, manually
audit the snippet so it contains no unbounded loops, recursion, blocking I/O or
large materialization.

## Related

- `B-python-execute-reentrant-gc-crash` — covers nested GC through Python,
  including this surface, but not unbounded execution.

## History
- `#1-source-scan-unbounded-query` `OPEN` reporter — Source-only scan confirmed unrestricted caller code reaches synchronous `ExecPythonCommandEx` on the main thread and that the only bound is a post-execution result-row cap. No build, test, editor, MCP call, or plugin edit was performed.
