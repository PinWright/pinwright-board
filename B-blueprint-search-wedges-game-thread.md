---
id: B-blueprint-search-wedges-game-thread
title: "`blueprint.search` monopolises the game thread for the whole Find-in-Blueprints index and killed the editor on a large project"
status: DONE
severity: Critical
category: bug
tags: [blueprint-search, find-in-blueprints, tick-safety, editor-crash]
---

# `blueprint.search` monopolises the game thread and can kill the editor

Reproduced live, UE 5.8, plugin `d195a55d`, host project with a large `/Game` (marketplace content).

A single first-of-session call — `blueprint.search {query:"BeginPlay", path:"/Game", autoIndex:true, limit:5}` —
did the following:

- auto-index pump completed in **30.9 s** (`LogTemp: blueprint.search: auto-index completed in 30.9s`)
- the request then ran on, mass-loading matched Blueprints: levels registered, landscapes created, and a
  static mesh built with an `18320 MB` memory estimate against a `9690 MB` limit
- the MCP stream hit its 120 s deadline and dropped; the ticket kept running
- `editor.status` from the socket I/O thread reported the game thread had not ticked for **136 s**
- at ~7 minutes the editor died: `Assertion failed: Callback.Baton == Baton`
  (`Runtime/Renderer/Private/VT/VirtualTextureProducer.cpp:266`)

**This is not the previously reported task-graph crash.** `grep -c RecursionGuard` over the run's log is
**0** — the `++Queue(QueueIndex).RecursionGuard == 1` assert fixed in `ad717fd6` (2026-07-29) did not recur.
This is a different, still-live failure with the same user-visible outcome.

## Mechanism

`Handlers/Blueprint/BlueprintIndexHandler.cpp:129-170` pumps the index in a tight loop on the calling
thread — the game thread, since `FRpcDispatcher::ProcessRequest` marshals every handler there:

```cpp
while (FiBManager.IsCacheInProgress())
{
    if (FPlatformTime::Seconds() - StartTime > TimeoutSeconds) { ...; return false; }
    FiBManager.Tick(0.0f);
    FPlatformProcess::Sleep(0.001f);
}
```

The editor therefore never runs a real frame for the entire index and search. Two consequences:

1. **The timeout is unenforceable.** Elapsed time is checked only *between* `Tick()` calls, and one `Tick()`
   can load a level and build a multi-GB static mesh. Bound: 60 s. Observed: past 7 minutes.
2. **Starving the game thread is plausibly what killed it.** Virtual Texture producer callbacks are serviced
   during normal game-thread tick. Blocking it for minutes while mass-loading VT-heavy assets is a credible
   cause of `Callback.Baton == Baton`. Stated as a hypothesis to test, not a conclusion — but if it holds,
   fixing the blocking also fixes the crash.

## The machinery to fix this already exists in the plugin

`asset.dump_folder` does exactly the right thing: `Ctx.StartJob` returns a ticket immediately, progress
frames stream over SSE, `system.job_status` polls it, `jobs.jsonl` records it, and it is one of the five
verbs that registers a real cancel callback. `blueprint.search` uses none of it.

The same file already contains the pattern: `fe08a040` added `DetachStreamSearchToCoreTicker`
(`BlueprintIndexHandler.cpp:36-66`) precisely so the stop-and-join would not run on the RPC stack.

Suggested shape: return a ticket, register a core-ticker delegate that calls `FiBManager.Tick(dt)` once per
frame, report `GetCacheProgress()` as the progress fraction (already computed, currently used only inside
the timeout warning), finish when `IsCacheInProgress()` goes false, then run the search. The editor stays
responsive, progress is visible, other RPCs stop queueing behind it, and it becomes cancellable.

## Two related timeout defects

- `PumpFiBIndexing(60.0)` is hard-coded at the call site (`:202`) and ignores the caller's
  `timeoutSeconds`, which only bounds the later `FStreamSearch` phase. A caller passing `timeoutSeconds:600`
  still gets an unreachable 60 s pump bound; one passing `5` still waits out the pump.
- `StopAndJoinStreamSearch`'s spin (`:26-29`) is unbounded — no timeout at all. `ContinueSearchQuery` calls
  `BlockSearchQueryIfPaused()` *before* its `WasStopped()` check
  (`FindInBlueprintManager.cpp:2914-2920`), so a FiB pause for GC or package-save can hang the game thread
  there indefinitely.

## Notes for whoever picks this up

The wedged-game-thread probe (`ca0a2fee`) worked exactly as designed here and is what made the diagnosis
possible: instead of a dead socket it named the RPC, the request id and the elapsed time, and correctly said
retrying was harmless. Keep that behaviour.

## History

- `#1-reported-with-live-repro` `OPEN` reporter — Confirmed at runtime on `d195a55d` / UE 5.8: one first-of-session `blueprint.search` wedged the game thread for 136 s+, dropped its own MCP stream at 120 s, and killed the editor at ~7 min with a VirtualTextureProducer assert. Zero `RecursionGuard` hits, so the previously reported task-graph crash is separately confirmed fixed.
- `#2-run-driven-from-core-ticker` `IN-REVIEW` developer — `blueprint.search` no longer does any waiting on the RPC stack. All three blocking sites removed in `Handlers/Blueprint/BlueprintIndexHandler.cpp`: the `IsCacheInProgress()` pump loop is gone (FiB is an `FTickableEditorObject`, so one `Tick(0.0f)` kick starts the caching op and the editor advances it — the handler only observes it); the `IsComplete()` spin is gone; and `GetParentBlueprint()` — a `LoadObject` per matched root, unbounded by `limit`, the actual cause of the mass asset loading and the VT assert — is replaced by two AssetRegistry hash lookups on the root result's object path. The run is a core-ticker state machine answering through `FAsyncResponseToken`, with no job ticket so the wire shape is unchanged. `timeoutSeconds` now bounds index-wait + search together (the hard-coded `PumpFiBIndexing(60.0)` is gone), default lowered 120 → 100 for headroom under the transport's 120 s response deadline, and it is enforceable because elapsed time is sampled once per frame instead of around calls that can load a level. `StopAndJoinStreamSearch` (`BlueprintIndexHandler.h/.cpp`) is now bounded (default 5 s) and falls back to `DetachStreamSearchToCoreTicker`, closing the unbounded `BlockSearchQueryIfPaused` spin. New disclosure fields: `indexInProgress`, `indexProgress`, `indexWaitSeconds`, `indexTimedOut`, `searchRan`. New tests in `Tests/Blueprint/TestBlueprintSearchGameThreadLiveness.cpp`: `PinWright.blueprint.search.AnswersOffTheRpcStack` (the handler must return WITHOUT having answered) and `PinWright.blueprint.search.EmptyScopeSkipsIndexWork` (an empty scope is decided before any FiB work). Not compiled or run — orchestrator builds.

- `#3-runtime-verified` `DONE` verifier — 2026-08-28, rebuilt DLL at plugin HEAD `b79ba53e`, editor pid 14932. Ran this ticket's verbatim repro under its original precondition: a grep of the live log confirmed **zero prior `blueprint.search` or Find-in-Blueprints traffic in the session**, so this really was the first-of-session cold-index call.
  `blueprint.search {query:"BeginPlay", path:"/Game", autoIndex:true, limit:5}` returned in ~26 s with 135 matching blueprints and `indexWaitSeconds: 25.09`, `searchElapsedSeconds: 1.07`, `timedOut: false`, `searchRan: true`, `indexInProgress: false`, `indexTimedOut: false`, `unindexedCount: 404`.
  **Game-thread liveness measured, not inferred.** A `register_slate_post_tick_callback` probe recorded every tick across the call: **1,922 ticks in 32.6 s, longest single stall 0.247 s.** Before: the game thread did not tick for 136 s+, the MCP stream dropped at its 120 s deadline, and the editor died at ~7 min on `Callback.Baton == Baton` (`VirtualTextureProducer.cpp:266`). None of that recurred — no VT assert, no mass asset loading, no level registration or landscape creation in the log, and the editor is the same pid afterwards. The auto-index now advances on the editor's own tick while the handler observes it, which is what the fix claimed.
  Source at this HEAD confirms the three blocking sites are off the RPC stack: the `IsCacheInProgress()` pump loop is gone (one `FiBManager.Tick(0.0f)` kick, then passive observation in `TickSearchRun`), `GetParentBlueprint()` exists nowhere in the plugin (replaced by AssetRegistry lookups, so no `LoadObject` per matched root), `PumpFiBIndexing` exists nowhere, and `timeoutSeconds` (default 100, clamped 1-600) drives one deadline across index-wait and search together. `blueprint.search` is deliberately **not** on `GTickUnsafeMethodNames` — it self-defers to the core ticker through its own `FAsyncResponseToken`, so it needs no table entry.
  **Two residuals, neither reopening this.** (a) One bounded `FPlatformProcess::Sleep(0.001f)` spin survives in `StopAndJoinStreamSearch` (`BlueprintIndexHandler.cpp:66-82`), but it is 5 s-bounded with a `DetachStreamSearchToCoreTicker` fallback and is reached only from the core ticker, never the RPC stack — so the unbounded `BlockSearchQueryIfPaused` hang this ticket named is closed. (b) Three of the five new disclosure fields are conditional, not unconditional: `indexProgress` only while the index is still building, `indexWaitSeconds` / `indexTimedOut` only when a wait happened, and the empty-scope early-out emits neither `searchRan` nor any of the three despite a comment there claiming the same response shape. Worth a one-line follow-up so callers can branch on presence.
