---
id: B-blueprint-search-wedges-game-thread
title: "`blueprint.search` monopolises the game thread for the whole Find-in-Blueprints index and killed the editor on a large project"
status: OPEN
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
