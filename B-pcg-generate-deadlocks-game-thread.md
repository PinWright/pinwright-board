---
id: B-pcg-generate-deadlocks-game-thread
title: "pcg.generate times out at 300s and reports failure while the generation actually completes"
status: IN-REVIEW
severity: High
category: bug
tags: [pcg, timeout, false-failure, async-job, ticket]
encounters: 1
lastSeen: 2026-08-12T00:00:00Z
---

# pcg.generate times out at 300s while the generation actually completes

The RPC hits the 300s completion timeout and returns a failure to the caller — **but the
generation completes afterwards anyway**. The caller therefore sees a hard failure for
work that in fact succeeded, and any subsequent logic that branches on that failure is
wrong. The result is unrecoverable: no ticket to poll, and the late completion resolves
into a request id the transport has already failed and dropped.

Repro (2026-08-12, UE 5.8, `EAContentExamples58`): call `pcg.generate` on a PCG component
whose graph does real work. Observe the RPC time out at 300s; then read the component's
point count on a later call and find the points present.

**Workaround (pre-fix):** drive `PCGComponent.generate(True)` from `python.execute` (fire
and forget) and read the resulting point count on a **later**, separate call.

## Confirmed root cause (source-verified 2026-08-13)

The original diagnosis — "blocks the game thread the scheduler needs" — is **wrong for the
shipped code**, and the correction matters because it changes the fix. The shipped handler
(`PCGGenerateHandler.cpp`, pre-fix :123) already returned immediately and resolved a single
`FAsyncResponseToken` from `OnPCGGraphGeneratedDelegate`; the game thread was never wedged,
and concurrent RPCs were never stalled. The real chain is:

1. PCG completion is signalled only from `UPCGComponent::PostProcessGraph` →
   `OnPCGGraphGeneratedDelegate.Broadcast` (`PCGComponent.cpp:803`), reached from the
   terminal task `UPCGSubsystem::ScheduleComponent` schedules
   (`PCGSubsystem.cpp:851-866`), executed by the graph executor inside
   `UPCGSubsystem::Tick` → `IPCGBaseSubsystem::Tick()` (`PCGSubsystem.cpp:421`) under a
   per-frame time budget. So completion is a game-thread, multi-frame, **unbounded**
   wall-clock wait whose duration tracks the editor's frame rate — an unfocused /
   throttled editor (the normal state in an agent session) stretches a seconds-long
   generation into minutes.
2. `Ctx.MakeAsyncToken()` converts that unbounded wait into one held-open HTTP request
   with **no job ticket**, so the caller has no `ticket_id` and cannot poll
   `system.job_status`.
3. `FSocketHttpServer::ProcessCompletionTimeouts` (`SocketHttpServer.cpp:1058-1091`) fires
   at the deadline — `MaxTimeoutMs` = 300s for a streaming request, `DefaultTimeoutMs` =
   120s otherwise (`SocketHttpServer.cpp:861-862`) — sends `Request timed out after 300s`
   with code `TIMEOUT`, and **removes** the pending completion. The generation then
   finishes and the token resolves into a dead request id; the answer is discarded.

The original hypothesis is still directionally right about one thing: a synchronous wait is
impossible. Because PCG only advances inside the game-thread tick, any in-handler wait
would be a genuine self-deadlock — so a "synchronous mode" was deliberately NOT added.

## Fix (IN-REVIEW)

`pcg.generate` converted from the single-token hold to the existing ticketed job seam
(`Ctx.StartJob` → `system.job_status`), in
`Source/PinWrightPCG/Private/Handlers/PCG/PCGGenerateHandler.cpp`:

- Kickoff returns `{status:"running", ticket_id, ...}` immediately (streaming clients still
  block-and-stream via the job-event bridge, but their progress frames carry the ticket, so
  a dropped/timed-out stream degrades to polling instead of losing the result). The ticket
  outlives the request (TTL 3600s), so the transport deadline can no longer destroy the
  answer.
- Completion funnels through one `Resolve()`: the generated delegate, the cancelled
  delegate (5.4+), and a 1s `FTSTicker` **watchdog** that closes the ticket from component
  state (`IsGenerating()` / `bGenerated` / generated output / `IsPartitioned()`) if no
  broadcast ever arrives, if the component is destroyed, or at the caller's
  `timeoutSeconds` cap. Where the state is ambiguous the watchdog resolves in favour of
  success, so the verb can no longer report failure for work that succeeded.
- Nothing-scheduled (`InvalidPCGTaskId`, which broadcasts nothing) is still answered
  **inline**, before any ticket exists — a caller is never handed a ticket that cannot move.
- `timeoutSeconds` (default 1800) caps only the job's wait and never cancels the
  generation; `system.job_cancel` does cancel it (wired via `SetCancelCallback` →
  `UPCGComponent::CancelGeneration`, 5.4+).
- New opt-in `cleanupFirst` (default false) schedules `CleanupLocal(bRemoveComponents=true)`
  before generating, which the generate task then depends on — the in-surface answer to the
  `cleanup()` trap below.

Tests (`Source/PinWrightPCG/Private/Tests/PCG/TestPCGGenerateHandler.cpp`):
`TicketedKickoffDoesNotBlock` (answer exists on the invoking stack, ticket registered under
method `pcg.generate`) and `AsyncDocs` (method page teaches ticket → `system.job_status`;
namespace page carries both traps). Docs: `docs/wiki-src/pcg.md` (`### pcg.generate` +
`## Engine-side traps when driving PCG`) and the ticket-pattern table in
`docs/wiki-src/system.md`.

## Related non-PinWright gotchas found alongside this (now documented on the pcg wiki page)
- `PCGComponent.cleanup()` must be called as `cleanup(True)`. Python binds
  `UFUNCTION Cleanup(bool bRemoveComponents)` and a missing argument defaults to `false` →
  `CleanupLocal(false)`, which keeps the generated components for reuse, so the next
  generate looks partially inert. Silent: the call is `void` and logs nothing. Cost one
  agent three tuning cycles of apparently-inert graph edits.
- `PointsPerSquaredMeter`: the field note ("inert unless `bUseLegacyGridCreationMethod` is
  on") is **inverted**. Source (`PCGSurfaceSampler.cpp:65-98`, `:149-159`): in the default
  non-legacy mode the density *is* read, but the derived cell size is floored at
  `2 × PointExtents` and every cell emits a point (`Ratio = 1.0`), so once extents dominate
  the knob does nothing; in legacy mode the density is only a thinning ratio clamped to ≤ 1
  (can remove points, never add). Either way `PointExtents` is the real spacing control,
  non-linearly (cell size ∝ √(1 / `PointsPerSquaredMeter`), floored by `2 × PointExtents`).

## History
- `#1-initial-repro` `OPEN` reporter — "pcg.generate wedges the game thread the PCG scheduler needs; RPC times out at 300s while the generation itself completes. Caller sees a false failure and every concurrent RPC stalls for the timeout window. Workaround: PCGComponent.generate(True) via python.execute, read point count on a later call."
- `#2-root-cause-corrected-and-ticketed` `IN-REVIEW` developer — "Root cause corrected from source: the handler never blocked the game thread (it already used MakeAsyncToken); it held the HTTP request open with no ticket while PCG completion advanced over game-thread ticks (PCGSubsystem.cpp:421 → PCGComponent.cpp:803), and SocketHttpServer.cpp:1058 failed the request at the 300s streaming deadline and dropped it. Fixed by converting the verb to Ctx.StartJob + system.job_status with a 1s watchdog ticker that closes the ticket from component state (favouring success when ambiguous), keeping the not-scheduled path inline, adding cleanupFirst/timeoutSeconds, wiring system.job_cancel to UPCGComponent::CancelGeneration, plus tests and wiki docs. NOT compiled and NOT runtime-verified by this agent (build owned by the integration agent, editor deliberately down): needs a live pcg.generate on a real graph to confirm ticket → completed with a non-zero pointCount."
- `#3-partially-verified-stays-in-review` `IN-REVIEW` tester — Built and runtime-verified **in part**.
  Status deliberately NOT advanced to `DONE`: the headline fix works, but the acceptance criterion
  `pointCount > 0` could not be met and cancellation could not be exercised.

  **VERIFIED — the actual fix.** `pcg.generate {actorName:"PW_PCGProbe", graphPath:"...", wait:false}`
  returns **synchronously** with `{"status":"running","ticket_id":"j_...","monitor_path":...}` — this is
  the change; pre-fix the call held the HTTP request open with no ticket. `system.job_status
  {ticket_id}` then reports `status:"completed"` with `completionSignal:"generated_delegate"`, i.e. the
  ticket is genuinely closed by `OnPCGGraphGeneratedDelegate` and the answer outlives the request.
  Reproduced across three graphs and two actors (a bare StaticMeshActor that the verb added a
  `UPCGComponent` to, and the example `BP_PCG_Grid`). Inline pre-ticket errors confirmed:
  bogus `actorName` → `ACTOR_NOT_FOUND`. The regenerated `pcg.generate` wiki page carries the full
  async contract, names `system.job_status`, and lists `cleanupFirst`/`timeoutSeconds`. All 6
  `PinWright.pcg.generate.*` tests pass, including the new `TicketedKickoffDoesNotBlock` and `AsyncDocs`.

  **NOT VERIFIED — `pointCount > 0`.** Every run reported `pointCount: 0, dataCount: 0` even though PCG
  demonstrably generated: Python inspection of the same components found `ISM_PCG_Cube_1` with **97
  instances** and `ISM_PCG_Cube_0` with **64 instances**, and `generated == True` on both
  `UPCGComponent`s. **This is NOT a regression from this fix.** The count arithmetic
  (`CountGeneratedPoints(Comp->GetGeneratedGraphOutput())`) is byte-identical to the pre-fix code —
  old `PCGGenerateHandler.cpp:140` → new `:227` — and `git diff` shows this commit added only two new
  `GetGeneratedGraphOutput().TaggedData.Num() > 0` uses, both in the new watchdog. The underlying cause
  is that on UE 5.8 `UPCGComponent::GetGeneratedGraphOutput()` is empty once generation completes, so
  anything reading it post-hoc sees nothing. Filed separately as
  `B-pcg-generated-graph-output-empty-after-generate`.

  **NOT VERIFIED — `system.job_cancel`.** Every example graph available in this project completed in
  5–10 ms wall clock, leaving no window in which to issue a cancel. The 5.4+
  `UPCGComponent::CancelGeneration()` seam is therefore compiled but unexercised.

  **Risk this raises for the new watchdog:** the watchdog's "did it produce output" signal is the same
  `GetGeneratedGraphOutput().TaggedData.Num() > 0` that is empty on 5.8. It was not reached in any of
  these runs (the delegate always won), but on a `state_poll` path it would read "no output" for a pass
  that produced instances. It resolves ambiguity in favour of success by design, so the damage is
  bounded, but this should be re-checked once the readback defect above is fixed.

  Committed as `a4a44281` — the code is sound and no worse than before on any axis; it simply cannot be
  called fully verified until a generation reports a real point count.
