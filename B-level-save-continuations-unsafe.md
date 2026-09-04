---
id: B-level-save-continuations-unsafe
title: "level.save and level.save_as run engine-pumping save continuations outside the safe-point and request guards"
status: IN-REVIEW
severity: High
category: bug
tags: [level, save, jobs, safepoint, reentrancy, render-flush]
encounters: 1
lastSeen: 2026-09-03T23:08:29+03:00
---

# Level-save job continuations bypass the safe-point and request guards

## What happens

Both save verbs preserve their asynchronous job contract by posting a nested
`AsyncTask(ENamedThreads::GameThread)` from `FJobBindArgs::BindNativeDelegate`:
`level.save` at
`Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:378-432` and
`level.save_as` at `:485-543`. The later tasks call `McpSafeLevelSave` at
`:401` and `:505`, after the original handler has returned. Neither continuation
is routed through `PinWrightSafePoint::RunAtSafePoint`, and neither method is in
`GTickUnsafeMethodNames` (`Source/PinWright/Private/Dispatch/SafePoint.cpp:56`).

The escaped body is not passive bookkeeping. `McpSafeLevelSave` synchronously
flushes rendering, sleeps on the game thread, calls
`FEditorFileUtils::SaveLevel`, and can repeat that sequence five times with
exponential sleeps
(`Source/PinWright/Private/Utils/AssetUtils.cpp:1211-1297`). In addition,
`level.save_as` directly flushes rendering, queues forced GC, and flushes again
on its request-guarded but safe-point-ungated handler stack, before even
validating `savePath`
(`LevelHandler.cpp:451-464`).

## Why it matters

The save and render flush can pump editor work after the dispatcher's active
request/reentrancy scope has been released. Concurrent RPC work can therefore
enter while the level-save stack is live, reopening the same reentrancy class
that has crashed sibling engine-pumping routes. The synchronous sleeps can also
stall the editor for up to 4.5 seconds on a full verification-failing retry
sequence, while each `FEditorFileUtils::SaveLevel` call itself remains unbounded.
Severity is High: the impact class is an editor crash or freeze, discounted
because no crash through these two save verbs is recorded.

## What should happen

Keep the job and its required deferred start, but route the engine-pumping save
phase through the retained-context `PinWrightSafePoint::RunAtSafePoint` shape
landed for cross-dispatched handlers. That route must keep the dispatcher's
active request scope until the continuation finishes. Move the `save_as`
preflight flush/GC into the same safe continuation (or remove it if the helper's
flush makes it redundant). Add structural coverage for both job continuations
and ordering coverage showing a queued RPC cannot enter during the save.

## Workaround

Run level saves only while the editor is otherwise idle and do not issue
concurrent RPCs until the job is terminal. This reduces exposure but does not
provide a safe-point guarantee.

## Related

- Catalog patterns `nested-gamethread-continuation-escapes-gates`,
  `tick-unsafe-engine-work-off-safe-point`, and
  `blocking-wait-starves-the-work-it-needs`.
- `B-nested-gamethread-marshal-defeats-tick-gate` — explicitly classified these
  two job marshals as a separate case whose async contract must be preserved.
- `B-nanite-rebuild-continuation-unsafe` — the existing sibling and fix shape
  for engine-pumping work inside a job continuation.

## Fix

The bug was valid, but the proposed response-bearing `RunAtSafePoint` fix was
not: that helper may execute inline and requires an `FSafePointResponder`, while
non-streaming requests must retain their immediate `status: running` job ticket.
Streaming requests continue suppressing that immediate response and wait for the
existing terminal job response. `PinWrightSafePoint::DeferJobToSafePoint` is now
the responder-free, always-deferred production path. Each `StartJob` bind captures
its `FHandlerContext` by value and calls the helper synchronously, so
`FHandlerContext::DeferActiveRequestToSafePoint` reserves the dispatcher's
request scope before the handler returns; standalone contexts fall back to the
same core-ticker deferral without creating an async response token.

The retained-deferral APIs also accept an owner-abandoned callback. If the
dispatcher/request context ends before the core-ticker callback, the save body
does not run and the existing job completion callback records a terminal
`SAVE_FAILED` result instead of leaving the ticket permanently running.

Both `level.save` and `level.save_as` are now exact entries in the tick-unsafe
method table. Their nested game-thread tasks were replaced by the retained
core-ticker continuation while preserving the weak-level check, progress event,
save and disk verification, registry rescan, completion values, the non-streaming
immediate ticket response, the unchanged streaming terminal-response behavior,
and the terminal job result shape. The `level.save_as` preflight render flush /
forced-GC request / render flush was narrowed after verifying that UE 5.8
`SaveMap` / `SaveLevel` do not request garbage collection. The full-purge request
is restored inside the retained continuation immediately before the save, while
the surrounding render flushes remain removed because `McpSafeLevelSave` flushes
immediately before each attempt.

Direct `ErrorCodes` adoption required removing the namespace alias and converting
all 58 raw `SendError(TEXT("..."))` sites in `LevelHandler.cpp` to existing
`ErrorCodes::ERR_*` constants. The two level-save wiki tests now assert both the
non-streaming immediate-ticket/poll contract and the streaming terminal-result
contract.

Structural coverage is extended in
`PinWright.infra.tick_safety.HandlerHazardsStayGated`, and production
retention/order coverage is added as
`PinWright.infra.dispatcher.JobSafePointContinuationRetainsRequestScope`;
dispatcher-teardown coverage is added as
`PinWright.infra.dispatcher.JobSafePointAbandonsWhenDispatcherEnds`.
Deliberate exclusions: this change does not alter `McpSafeLevelSave` retry or
persistence policy, job response fields, streaming response behavior, or the
synchronous `SaveLevel` implementation. Verification in this implementation pass
is source-only; no build, automation test, editor, or MCP run was performed.

## History

- `#1-filed-save-continuation-gap` `OPEN` reporter — Source-only pattern scan confirmed both deferred level-save bodies escape the request and safe-point guards, and that the shared helper performs render flushes, synchronous save, and bounded game-thread sleeps. Board-wide dedup found the two methods only in `B-nested-gamethread-marshal-defeats-tick-gate` as explicitly excluded follow-up work; the Nanite sibling was filed separately, but the level-save follow-up was not. No build, test, editor, MCP call, or Saved-file access was performed.
- `#2-retained-save-job-scope` `IN-REVIEW` developer — Added the responder-free always-deferred job helper in `SafePoint.h`, gated both level-save verbs in `SafePoint.cpp`, routed both `LevelHandler.cpp` save binds through retained dispatcher scope, removed the redundant Save-As preflight, extended `PinWright.infra.tick_safety.HandlerHazardsStayGated`, and added `PinWright.infra.dispatcher.JobSafePointContinuationRetainsRequestScope`; source-authored only, not compiled or run.
- `#3-restored-full-gc-contract` `IN-REVIEW` developer — Restored the Save-As full-purge GC request inside the retained continuation, removed the error-code namespace alias and converted all 58 raw `SendError` sites to direct registry constants, extended `PinWright.infra.wiki_handler.Method.LevelSaveAsync` and `PinWright.infra.wiki_handler.Method.LevelSaveAsAsync` for both response modes and retained core-ticker safe-point wording, and corrected the wiki text. The teardown test omitted from history #2, `PinWright.infra.dispatcher.JobSafePointAbandonsWhenDispatcherEnds`, is also part of this fix. Source-only follow-up; no build, automation test, editor, or MCP run was performed.
