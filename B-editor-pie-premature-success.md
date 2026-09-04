---
id: B-editor-pie-premature-success
title: "`editor.play` and `editor.stop` return success when the PIE lifecycle request is only queued for a later tick"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, pie, async, lifecycle, false-success]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# PIE lifecycle verbs claim completion at request time

## What happens

`editor.play` calls `RequestPlayInEditorSession` and immediately sends
`success:true` (`PIEHandler.cpp:104-130`). The helper itself states that
`RequestPlaySession` only queues startup for the next tick
(`PieControlUtils.h:63-65`, `:172-173`). No delegate or world-state check confirms
that any requested client, net mode, or emulation session started.

`editor.stop` similarly calls `RequestEndPlayMap` and immediately sends success
(`PIEHandler.cpp:150-159`). The same tree already documents and handles this engine
contract correctly for `editor.quit`: stop is asynchronous and must be polled with
a timeout (`EditorQuitHandler.cpp:74-100`).

## Why it matters

Automation can issue its next input or capture against edit mode after `play`, or
mutate/save while PIE is still tearing down after `stop`. The response gives no
queued state or terminal job to distinguish that race. Severity is High for a
normal-path premature success receipt.

## What should happen

Return a job ticket and complete it only after the requested PIE world contexts are
active, or after no PIE session remains for stop. Use delegates or a bounded ticker
poll, propagate startup/teardown failure, timeout and cancellation, and report the
actual client/net state. If the API intentionally remains fire-and-forget, return
`requested:true`/`queued:true` instead of terminal success.

## Workaround

Poll `editor.pie_status` after either verb and do not continue until the requested
terminal state is observed.

## Related

- `B-editor-quit-crash-pie-active` — contains the existing bounded PIE-stop wait.
- `B-ui-stop-play-exec-noop` — different UI namespace path.

## Fix

The root cause was that both verbs treated the engine's deferred request API as a completed lifecycle transition. `Utils/PieState` now measures `PlayWorld`, Unreal's authoritative `IsPlaySessionInProgress()` state, and queued-start state. A small editor-thread lifecycle owner gives every waiter a generation: overlapping operations are rejected, while `editor.stop` explicitly invalidates an owned play waiter. Cancellation is evaluated before target success, so the old call returns registered `PIE_START_CANCELLED` exactly once without making any claim about whether a later `PlayWorld` is visible. `editor.play` still succeeds at its owned `PlayWorld` threshold rather than waiting for every multiplayer context.

`editor.stop` also snapshots state before touching Unreal. An existing `PlayWorld` selects `RequestEndPlayMap`; `endPlayRequested` directly mirrors the current `ShouldEndPlayMap()` flag. A no-world request is cancelled only while Unreal still reports it queued. When only `PlayInEditorSessionInfo` remains, cancellation is unsafe because UE 5.8's `CancelRequestPlaySession()` resets that state while deferred callbacks may still use it, so the decision seam selects a wait-only transition. The stop state probe requests end once a late `PlayWorld` appears, then completes only after both the authoritative session predicate and `PlayWorld` are inactive. Payloads distinguish queued cancellation from the non-cancellable state with `transitionCleanupPending`.

Play-start timeout follows the same rule: it reports `PIE_START_FAILED` immediately, cancels only a still-queued request, and otherwise retains the generation as a lifetime-safe second bounded cleanup watcher. That watcher requests end if `PlayWorld` appears, rejects conflicting lifecycle calls while pending, clears ownership on inactivity or cleanup timeout, and checks its generation before every engine action so stale cleanup cannot affect a later request. A redundant stop/cleanup waiter is rejected with registered `PIE_STOP_IN_PROGRESS`; immediate post-request/cancel states still respond through the live handler context before an async token is created.

The verifier follow-up adds dispatcher ownership without retaining the dispatcher's single-request serialization guard. Both lifecycle handlers acquire a lifetime-only lease, and dispatcher teardown cancels the exact wait ticker before releasing only that operation generation and returning `PIE_START_FAILED` or `PIE_STOP_FAILED` once. Normal completion unregisters the lease. The token-free timeout cleanup watcher reuses the same lease and owned wait control, so teardown cancels it without sending a second response after the original timeout. Subsystem deinitialization now quiesces and synchronizes the transport's request handoff before destroying the dispatcher. The I/O thread stays alive while typed abandonment and generic shutdown responses are resolved. Quiesced connections continue fixed-budget read/discard passes without parsing new requests, flush their response, send FIN, and complete the existing idle/absolute-capped linger phase before the drain reports completion. Accept and read loops observe shutdown gates, and the drain timeout is bounded by the 30-second absolute linger cap plus one second; queued game-thread dispatches validate a weak dispatcher lifetime instead of retaining raw access.

Files changed: `Source/PinWright/Private/Dispatch/RpcDispatcher.h`, `Source/PinWright/Private/Dispatch/RpcDispatcher.cpp`, `Source/PinWright/Private/Handlers/HandlerContext.h`, `Source/PinWright/Private/Handlers/HandlerContext.cpp`, `Source/PinWright/Private/Utils/PieState.h`, `Source/PinWright/Private/Utils/PieState.cpp`, `Source/PinWright/Private/Handlers/Editor/PIEHandler.cpp`, `Source/PinWright/Private/PinWrightSubsystem.cpp`, `Source/PinWright/Private/Transport/SocketHttpServer.h`, `Source/PinWright/Private/Transport/SocketHttpServer.cpp`, `Source/PinWright/Private/Handlers/ErrorCodes.h`, `Source/PinWright/Private/Tests/EditorOps/TestEditorPieLifecycle.cpp`, and `Docs/wiki-src/editor.md`.

Tests added: `PinWright.editor.pie.LifecycleWaitStateTransitions` uses shared-owned fake state (safe even if a ticker remains pending) and proves a cancelled stale waiter completes once even when a later world appears on the same tick. `PinWright.editor.pie.LifecycleDecisionSeams` deterministically covers pending/reached/timed-out/cancelled priority, active-world end requests, queued-start cancellation, session-info-only waiting, overlapping ownership rejection, and generation replacement. `PinWright.editor.pie.LifecycleHandlersUseBoundedWait` ratchets snapshot-before-action ordering, queued-state recheck, late-`PlayWorld` teardown wiring, bounded timeout cleanup ownership, stale-generation action guards, direct `ShouldEndPlayMap()` reporting, synchronous-response-before-token ordering, quiesce-before-dispatcher and drain-before-I/O-stop ordering, bounded read/discard, stop-aware accept/read loops, and FIN/linger-aware drain completion. `PinWright.editor.pie.PlayWaitAbandonsWhenDispatcherEnds` calls the production `editor.play` registration through a test dispatcher, proves the wait does not hold request serialization, then destroys the dispatcher and checks exactly one typed failure, immediate ticker invalidation, and no late overwrite.

Deliberately not changed: play success is still bounded by the requested `PlayWorld` appearing, not by every multiplayer client/context becoming ready; that broader contract exceeds this ticket. `editor.quit` keeps its separate exit-specific teardown ordering and 15-second failure path. No live PIE, build, or automation run was performed in this source-only worker.

## History
- `#1-source-scan-pie-receipt` `OPEN` reporter — Source-only scan matched both immediate success responses to engine APIs explicitly documented in this checkout as deferred to later ticks. No build, test, editor, MCP call, or plugin edit was performed.
- `#2-bounded-pie-lifecycle-wait` `IN-REVIEW` developer — Added bounded, generation-owned lifecycle waits. Stop invalidates an older play waiter before replacing it; cancellation wins over a later `PlayWorld`. Active world requests end, queued/no-world state may cancel only after a live queued-state recheck, and session-info-only/no-world state waits without engine cancellation until `PlayWorld` can be ended or the bound expires. Play timeout keeps a second bounded cleanup generation for that transition, blocks conflicts, and clears on inactivity or cleanup timeout. Pure decision and structural tests cover the transition action, late-world teardown, direct end-request reporting, and stale-generation guards.
- `#3-dispatcher-lifetime-return` `OPEN` verifier — Returned the fix because the 15-second PIE waiters retained only async response tokens: dispatcher teardown could neither cancel their tickers nor terminal-fail their requests, and existing coverage did not exercise destruction through the real handler registration.
- `#4-dispatcher-owned-pie-waits` `IN-REVIEW` developer — Added lifetime-only dispatcher leases for both PIE waiters without holding request serialization, cancellable ticker controls with exactly-once invalidation, typed teardown failures, and teardown ownership for the token-free timeout cleanup watcher. Added `PinWright.editor.pie.PlayWaitAbandonsWhenDispatcherEnds` through the production registration and kept `EditorCommandHandler.cpp` unchanged.
