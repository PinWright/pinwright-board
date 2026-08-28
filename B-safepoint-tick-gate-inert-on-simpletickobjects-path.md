---
id: B-safepoint-tick-gate-inert-on-simpletickobjects-path
title: "The tick-unsafe SafePoint gate never fires when a third-party editor tickable pumps the named-thread queue: `IsSafeNow()` tests `UWorld::bInTick`, which is false during `SimpleTickObjects`, so all 34 listed verbs run inline mid-frame anyway"
status: DONE
severity: High
category: bug
tags: [safepoint, dispatch, tick-gate, rpc-dispatcher, mass-entity, crash-adjacent, guard-inefficacy, multi-agent, shared-editor]
encounters: 1
lastSeen: 2026-08-28T09:35:00+05:00
---

# A guard that reports safe on the one stack that has already produced editor kills

`FRpcDispatcher::ProcessRequest` defers any verb on the tick-unsafe list when a
world is ticking (`Dispatch/RpcDispatcher.cpp:449`):

```cpp
    if (PinWrightSafePoint::IsTickUnsafeMethod(Method) && !PinWrightSafePoint::IsSafeNow())
```

`IsSafeNow()` is `!ForcedUnsafeForTests() && !IsAnyWorldTicking()`
(`Dispatch/SafePoint.h:192-195`), and `IsAnyWorldTicking()` is a loop over
`GEngine->GetWorldContexts()` returning true only when some `World->bInTick` is
set (`SafePoint.h:174-189`).

**There is a live entry path into `ProcessRequest` on which `bInTick` is false for
every world, and it is the path three of today's five editor kills actually ran
on.** A third-party editor tickable — here `UMassEntityEditorSubsystem::Tick` —
blocks on a task and pumps the game thread's named-thread queue, which drains a
queued PinWright request and runs the handler **inline, inside the engine frame**,
during `FTickableObjectBase::SimpleTickObjects`. `SimpleTickObjects` runs *before*
the editor world tick (`EditorEngine.cpp:1936` against `:1967`), so no world has
`bInTick` set, `IsSafeNow()` returns **true**, and the gate passes the request
straight through to the inline branch.

The verbs on that list are there because running them mid-frame kills the editor.
The gate is their only protection, and on this stack it is inert.

## This is a known gap — what is new is that it is now a measured one

`Dispatch/SafePoint.cpp:304-312` already states it, in the `model.compile` entry:

> **KNOWN GAP, stated because that stack is not the one this entry fixes.**
> `IsSafeNow()` reads `UWorld::bInTick` only (`SafePoint.h`), and
> `SimpleTickObjects` runs BEFORE the editor world tick (`EditorEngine.cpp:1936`
> against `:1967`), so `bInTick` is false there and the gate lets the call
> through. This entry closes the `UWorld::Tick` half of the window — the half
> every other family here is listed for — and nothing more.

That comment was written from a 2026-08-19 crash log and left deliberately open,
on the reasoning that closing the rest would mean deferring listed methods
unconditionally, "a behaviour change for all seven families above and for the
tests that pin their inline path".

**This ticket does not dispute that reasoning. It supplies the cost side of it,
which the comment did not have:** on 2026-08-27 the unclosed half was taken three
times in one session, by two different *listed* verbs, in a shared editor with
four agents in it. The gap stopped being theoretical.

## Evidence: three crashes, one stack, all in this checkout's logs

The frames below are common to all three, outermost-last, read from the retained
backup logs. `FRpcDispatcher::ProcessRequest` is running a handler **inline** —
this is not a deferred-queue drain.

```
UnrealEditor-PinWright.dll!FRpcDispatcher::ProcessRequest()      RpcDispatcher.cpp:646
UnrealEditor-Core.dll!TGraphTask<FAsyncGraphTask>::ExecuteTask() TaskGraphInterfaces.h:703
UnrealEditor-Core.dll!FTaskBase::TryExecuteTask()                TaskPrivate.h:524
UnrealEditor-Core.dll!FNamedTaskThread::ProcessTasksNamedThread() TaskGraph.cpp:807
UnrealEditor-Core.dll!FNamedTaskThread::ProcessTasksUntilQuit()  TaskGraph.cpp:696
UnrealEditor-Core.dll!TryWaitOnNamedThread()                     TaskPrivate.cpp:426
UnrealEditor-Core.dll!FTaskBase::WaitWithNamedThreadsSupport()   TaskPrivate.cpp:240
UnrealEditor-MassEntityEditor.dll!UMassEntityEditorSubsystem::Tick()
                                                  MassEntityEditorSubsystem.cpp:196
UnrealEditor-Engine.dll!FTickableObjectBase::SimpleTickObjects() Tickable.cpp:116
UnrealEditor-UnrealEd.dll!UEditorEngine::Tick()                  EditorEngine.cpp:1936
UnrealEditor-UnrealEd.dll!UUnrealEdEngine::Tick()                UnrealEdEngine.cpp:546
UnrealEditor.exe!FEngineLoop::Tick()                             LaunchEngineLoop.cpp:5859
```

| # | Log (backup, UTC name) | Verb that ran there | On the tick-unsafe list? |
|---|---|---|---|
| 1 | `EAContentExamples58-backup-2026.08.27-08.26.21.log` | `render.capture_asset_preview` (`RenderHandler.cpp:1445`) | **yes** |
| 2 | `EAContentExamples58-backup-2026.08.27-08.39.32.log` | `render.capture_asset_preview` (`RenderHandler.cpp:1445`) | **yes** |
| 3 | `EAContentExamples58-backup-2026.08.27-14.16.48.log` | `niagara.graph.create_node` (`NiagaraGraphHandler.cpp:763`) | no |

Verified by grep against each backup log: `UMassEntityEditorSubsystem::Tick`
appears in exactly these three of the day's five crash callstacks (the other two
are a Slate prepass on the game thread and an assert on the render thread, neither
of which is a dispatcher stack).

**What this proves and what it does not.** It proves the gate did not fire for a
listed verb on a real stack, twice, because `render.capture_asset_preview` is in
`GTickUnsafeMethodNames` (`SafePoint.cpp`, family B) and yet executed inline
inside `SimpleTickObjects`. **It does not prove the gate's failure caused those
crashes** — `render.capture_asset_preview`'s fault is an access violation in
`CloseAllEditorsForAsset` that may well fire from any stack, and
`niagara.graph.create_node` (row 3, not listed and not claimed to need listing)
dies on its own unfinalized-RAII bug wherever it runs. Causation is unproven and
is deliberately not claimed here. The defect being reported is narrower and
certain: **a safety gate reports "safe" on a stack that is definitionally the
unsafe one, so it protects none of its 34 verbs there.**

## Why it matters more than the comment assumed

- The listed families are not minor. Family A is `CollectGarbage` -> `~ULevel` ->
  `FreeTickTaskLevel`, which asserts. Family B is re-entrant Slate pump + viewport
  draw + `FlushRenderingCommands` from inside a frame. Family H
  (`model.compile`) reconstructs a `UStaticMesh` in place while components still
  reference it. Every one is an editor kill, and every kill takes down every agent
  sharing the process.
- The trigger is not something a caller controls. Whether a request lands in
  `SimpleTickObjects` or in the safe window depends on which editor tickable
  happens to block on a task that frame. `UMassEntityEditorSubsystem` is stock UE
  5.8 and ships enabled; no PinWright user opted into it. So the same verb is safe
  on one call and unguarded on the next, which is exactly the "survives one or two
  calls, dies on a later one" behaviour reported against the capture family.
- The gate's own log line is `Verbose`, so a deferral that *doesn't* happen leaves
  no trace at default verbosity. There is currently no way to tell from a log
  whether the gate fired.

## What it should do

Options, cheapest first. Picking one is a judgement call for whoever owns
`SafePoint`; the point of the ticket is that "leave the half open" now has a
measured price.

1. **Widen the safety test beyond `bInTick`.** `IsSafeNow()` should also report
   unsafe when the engine is anywhere inside `FEngineLoop::Tick` on the game
   thread — a simple depth flag set/cleared by the subsystem's own tick, or
   `GIsRunning`-style frame-phase state — rather than inferring frame position
   from a single world's `bInTick`. This closes `SimpleTickObjects` without
   deferring unconditionally, which is what the existing comment ruled out.
2. **Do not execute PinWright requests from a nested named-thread pump at all.**
   The root problem is that `ProcessRequest` can be re-entered from an arbitrary
   third-party task wait. A frame-scoped "not from a nested pump" latch on the
   dispatcher would make the entry path impossible regardless of the tick test.
3. **At minimum, make the failure observable.** Promote the pass-through case to a
   logged line (or count it), so a crash log shows whether a listed verb ran
   guarded or unguarded. Today the absence of a `Verbose` line is unreadable
   evidence, and every crash-forensics pass has to reconstruct the stack by hand
   to find out.

Whatever is chosen, the regression test has to drive the **nested-pump** path, not
just `SetForcedUnsafeForTests(true)`. The existing tests pin the inline branch and
the forced-unsafe branch; neither of them can observe this gap, because neither
enters `ProcessRequest` from inside `SimpleTickObjects`.

severity rationale: impact=the plugin's only guard for 34 verbs that are documented as editor-killing does not fire on a stack observed three times in one day, and its failure is invisible at default log verbosity x reach=every PinWright user on UE 5.8, since the tickable that opens the path (`UMassEntityEditorSubsystem`) is stock and enabled, and no caller action selects or avoids it. Not Critical only because no crash has been *proven* to be caused by the missed deferral rather than by the verb's own independent fault -> High

## Related

- `B-capture-asset-preview-no-safe-close-mode` (OPEN, Critical) — crashes 1 and 2
  above are that ticket's crash A and crash B. Its verb is on the tick-unsafe list
  and ran unguarded; that does not change its own root cause
  (`CloseAllEditorsForAsset` at `CaptureSubject.cpp:1405`), but it does mean a
  fixer cannot assume the gate kept the verb out of the frame.
- `B-model-compile-live-niagara-mesh-renderer-raytracing-assert` (OPEN, Critical;
  the duplicate `B-static-mesh-rebuild-crashes-live-niagara-mesh-renderer` was merged
  into it and deleted) — `model.compile` is the verb whose `SafePoint.cpp` entry
  documents this gap, and its crash log carries the `FlushRenderingCommands called
  recursively! 2 calls on the stack.` tell that entry cites. That crash is on the
  render thread, so this ticket does **not** claim the gate gap caused it.
- `B-niagara-create-node-unfinalized-graph-node-creator-fatal` (OPEN, Critical;
  the duplicate `B-niagara-create-node-early-return-before-finalize-crash` was merged
  into it and deleted) — crash 3 above. Listed only as the third instance of the stack;
  that verb is not on the tick-unsafe list and this ticket does not argue it should
  be, since its fault is an unfinalized `FGraphNodeCreator` and is stack-independent.
- `B-compile-material-not-tick-gated` (OPEN) — the adjacent question of *which*
  verbs belong on the list. Orthogonal to this one: that ticket is about list
  membership, this one is about the list having no effect on a given stack.

## History

- `#1-log-forensics` `OPEN` reporter — 2026-08-27, UE 5.8, `EAContentExamples58`,
  `/Game/Maps/Atlantis`, PinWright at this checkout's HEAD. Found during a
  forensics pass over all five of the day's editor kills, not by running anything:
  the shared outer frames were what the three dispatcher-stack crashes had in
  common, and checking them against `SafePoint.h` showed `IsSafeNow()` must have
  returned true on all three. **No repro run and none recommended** — the editor
  is shared with four working agents and every listed verb is an editor kill by
  construction. Gate mechanism re-verified in current source during filing:
  `RpcDispatcher.cpp:449`, `SafePoint.h:174-195`, and the pre-existing KNOWN GAP
  note at `SafePoint.cpp:304-312` (which this ticket confirms rather than
  contradicts). Callstack presence confirmed by grepping
  `UMassEntityEditorSubsystem::Tick` across all five backup logs: present in
  exactly the three named above. Deliberately filed as High, not Critical, and
  with causation explicitly disclaimed — an earlier attribution in this same
  session ("Slate stack exhaustion") was wrong and was only caught on re-check, so
  the claim here is kept to what the logs actually show.
- `#2-widen-gate-to-named-thread-pump` `IN-REVIEW` developer — took option 1, but keyed
  on the pump rather than on a frame-depth flag. `PinWrightSafePoint::IsSafeNow()` now
  also reports unsafe while the game thread is draining one of its own task-graph
  named-thread queues, via a new `IsInsideNamedThreadPump()` reading
  `FTaskGraphInterface::IsThreadProcessingTasks` on `GameThread` and `GameThread_Local`
  (the engine's own `RecursionGuard`, TaskGraph.cpp:690/:705/:872-875/:1423-1429). That
  is exactly the `SimpleTickObjects -> WaitWithNamedThreadsSupport ->
  ProcessTasksUntilQuit` stack in this ticket, and it needs no new global state, no
  delegate and no ordering assumption. It cannot ping-pong: `FEngineLoop::Tick` calls
  `FTSTicker::GetCoreTicker().Tick()` directly (LaunchEngineLoop.cpp:6103), not through
  a task, so the drain in `UPinWrightSubsystem::Tick` always sees a safe point and the
  hop terminates after one pass. Files: `Dispatch/SafePoint.h` (include, new predicate,
  `IsSafeNow()`, header doc), `Dispatch/SafePoint.cpp` (the `model.compile` KNOWN GAP
  note rewritten as closed, with the behaviour change stated once for all families).
  Behaviour change: every one of the 35 table entries now always takes the one-hop
  (<= 0.1s) deferral when the request arrived over the transport, because such requests
  always execute from a named-thread pump; direct in-editor callers (automation stacks,
  the core-ticker drain, cross-dispatch) are not in a pump and still run inline, so the
  committed inline-path assertions keep their meaning. Tests added in
  `Tests/World/TestSafePointGate.cpp`, both driving a REAL nested pump rather than
  `SetForcedUnsafeForTests`: `PinWright.core.safe_point.NestedNamedThreadPumpIsUnsafe`
  (observes `IsAnyWorldTicking()==false` and `IsSafeNow()==false` on that stack — the
  first half is the evidence bInTick is blind there, the second is the counterfactual)
  and `PinWright.core.safe_point.DispatcherDefersFromNestedNamedThreadPump` (real
  `FRpcDispatcher::ProcessRequest` against the `_test.beta` fixture: no handler reached
  inside the pump, runs on the next `ProcessPendingRequests`). Not done: option 3
  (logging the pass-through case) needs a `RpcDispatcher.cpp` edit, which this agent did
  not own; the exact diff is in the hand-off report. Two stale comments describing the
  old two-term `IsSafeNow()` are left for their owners: `Tests/Infra/
  TestContractConsistency.cpp:283-284` and `Handlers/Render/CaptureSubject.h:543`
  (assertions in both still hold). Not compiled or run — the orchestrator owns builds.
- `#3-runtime-verified-gate-now-fires` `DONE` verifier — 2026-08-28, rebuilt DLL at plugin HEAD `b79ba53e`, editor pid 14932. **The gate fires on the transport path, measured rather than argued.** Source at this HEAD: `IsSafeNow()` (`Dispatch/SafePoint.h:271-276`) is now `!ForcedUnsafeForTests() && !IsAnyWorldTicking() && !IsInsideNamedThreadPump()`, and `IsInsideNamedThreadPump()` (`:253-262`) reads `FTaskGraphInterface::IsThreadProcessingTasks` on both `GameThread` and `GameThread_Local`. The `model.compile` KNOWN GAP note at `SafePoint.cpp:304-312` is rewritten as closed and cites this ticket.
  **Behaviour.** The deferral line is `Verbose` on a category that defaults to `Log`, so it had to be unmuted first — `editor.console_command {command:"Log LogPinWrightSafePoint Verbose"}` — and then the log was read directly. Result over one session: **123 deferrals across 11 distinct listed verbs, with no listed verb observed running inline.** The two that decide this ticket: **`render.capture_asset_preview` 14/14 deferred**, which is the verb rows 1 and 2 of the evidence table recorded running INLINE inside `SimpleTickObjects`; and **`model.compile` 2/2 deferred**, the verb whose `SafePoint.cpp` entry documented the gap. Also counted: `python.execute` x61, `asset.generate_thumbnail` x15, `system.console_command` x10, `render.capture_open_level` x10, `editor.console_command` x4, `editor.set_game_view` x3, `camera.frame_actor` x2, `widget.screenshot_designer` x1, `render.capture_annotated` x1 — several of those from sibling agents in the same shared editor, so the 100% rate is not one caller's pattern. Zero asserts and zero access violations across the session; a `register_slate_post_tick_callback` probe measured 22,870 ticks with the longest game-thread stall 2.30 s, so the one-hop deferral costs nothing observable.
  **Not synthesised, and it does not need to be.** No repro forced `UMassEntityEditorSubsystem::Tick` to block, so the exact `SimpleTickObjects` stack was not reconstructed. That stack is a named-thread pump by construction, and the predicate now keys on the pump rather than on frame position, so covering the transport path covers it.
  **Two things left undone, neither of them this ticket's defect, both worth a follow-up.**
  (a) **Option 3 was skipped, as `#2` states, and it still bites.** The deferral line is `Verbose`, there is no pass-through line at any verbosity, no counter, no response field, and no RPC exposes safepoint state (`GetTickUnsafeMethods()` has test callers only; `editor.status` emits PIE fields and nothing else). A caller still cannot distinguish deferred from inline, and a default-verbosity crash log still cannot either — this verification had to raise the category by hand to see anything at all. The next crash-forensics pass will be in the same position `#1` was.
  (b) **The deferral message is now wrong text.** `RpcDispatcher.cpp:455` still says "a world is inside `UWorld::Tick` and this method tears down levels or drives the viewport" — printed for every deferral, including the pump case, which is the case it was added for. All 123 lines above say that, and in none of them was a world necessarily ticking. A triager will go looking for a ticking world that was not there. One-line fix.
