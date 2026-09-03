---
id: B-level-save-continuations-unsafe
title: "level.save and level.save_as run engine-pumping save continuations outside the safe-point and request guards"
status: OPEN
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
on its initial ungated handler stack, before even validating `savePath`
(`LevelHandler.cpp:451-464`).

## Why it matters

The save and render flush can pump editor work after the dispatcher's active
request/reentrancy scope has been released. Concurrent RPC work can therefore
enter while the level-save stack is live, reopening the same reentrancy class
that has crashed sibling engine-pumping routes. The synchronous sleeps can also
stall the editor for over seven seconds on a full retry sequence. Severity is
High: the impact class is an editor crash or freeze, discounted because no crash
through these two save verbs is recorded.

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

## History

- `#1-filed-save-continuation-gap` `OPEN` reporter — Source-only pattern scan confirmed both deferred level-save bodies escape the request and safe-point guards, and that the shared helper performs render flushes, synchronous save, and bounded game-thread sleeps. Board-wide dedup found the two methods only in `B-nested-gamethread-marshal-defeats-tick-gate` as explicitly excluded follow-up work; the Nanite sibling was filed separately, but the level-save follow-up was not. No build, test, editor, MCP call, or Saved-file access was performed.
