---
id: B-recorder-query-mainthread-freeze
title: "`recorder.query` executes unrestricted caller Python synchronously on the editor main thread with no timeout or cancellation"
status: IN-REVIEW
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

## Fix

Root cause: `recorder.query` passed unrestricted caller code directly to the editor's embedded Python interpreter through synchronous `ExecPythonCommandEx`, so `maxRows` could only apply after a loop returned. The handler now writes the same wrapper to a temporary file, launches it in the engine-provided isolated Python child process, tracks it with `FHandlerContext::StartJob`, and drains output on the core ticker. Timeout and cancellation dispatch process-tree termination to a background worker holding a pre-opened process handle; the run remains alive until worker cleanup is safe. Timeout results expose `timedOut: true`, and `terminationConfirmed` is true only when both child exit and worker completion are observed before the hard cutoff. If the cutoff wins, `cleanupIncomplete: true` reports the bounded cleanup state without claiming eventual cleanup.

Files changed:
- `Plugins\PinWright\Source\PinWright\Private\Handlers\Recorder\RecorderQueryHandler.cpp`
- `Plugins\PinWright\Source\PinWright\Private\Handlers\ErrorCodes.h`
- `Plugins\PinWright\Source\PinWright\Private\Tests\Recorder\TestRecorderQuerySafety.cpp`
- `Plugins\PinWright\Docs\wiki-src\recorder.md`

Test IDs:
- `PinWright.recorder.query.DeclaresBoundedTimeout` — real dispatcher timeout job with a 1-second sleep and `timeoutSeconds: 0.5`; latent registry assertion checks terminal `timedOut: true`.
- `PinWright.recorder.query.ResolvesSessionsUnambiguously` — real dispatcher ambiguity/error candidate assertions and exact-filename query completion.

Deliberately not changed: the pure-stdlib `Content\Python\recorder_query.py` surface and the shared structured-verb `RecorderResolver` behavior. No live editor, compile, or automation run was performed.

## History
- `#1-source-scan-unbounded-query` `OPEN` reporter — Source-only scan confirmed unrestricted caller code reaches synchronous `ExecPythonCommandEx` on the main thread and that the only bound is a post-execution result-row cap. No build, test, editor, MCP call, or plugin edit was performed.
- `#2-bounded-query-job` `IN-REVIEW` developer — Replaced synchronous embedded execution with a cancellable child-process job bounded by `timeoutSeconds` and a 60-second hard maximum, with deadline-first classification, exit-confirmed cleanup, and explicit `terminationConfirmed` reporting for the bounded termination phase. Static source/registration ratchets were added. No compile or test run was performed.
- `#3-worker-owned-cleanup-and-behavior-tests` `IN-REVIEW` developer — Corrected UE 5.8 mutable process-handle usage, moved recursive process-tree termination to a worker-owned pre-opened handle, kept the run alive through cleanup, and added `cleanupIncomplete` for the hard cutoff. Replaced source-grep checks with real dispatcher/job behavior tests. No compile or test run was performed.
