---
id: E-safepoint-inline-run-leaves-no-trace
title: "A tick-unsafe verb that runs INLINE logs nothing, so a crash log cannot say whether the SafePoint gate fired"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [safepoint, dispatch, observability, rpc-dispatcher, crash-triage, tick-gate]
encounters: 1
lastSeen: 2026-08-28
---

# The gate says when it defers and is silent when it does not

`FRpcDispatcher`'s tick-unsafe gate logs a Verbose line when it **defers** a request. When
`IsSafeNow()` returns true and the verb runs **inline**, it logs nothing.

That asymmetry matters precisely when it is most expensive. Only the 35 table entries reach that
branch, and they are the verbs that tear down levels or drive the viewport — the ones a crash is most
likely to involve. After an editor death, the log can prove a deferral happened but cannot distinguish
"the gate correctly let this run" from "the gate did not fire when it should have". That is the
question `B-safepoint-tick-gate-inert-on-simpletickobjects-path` existed to answer, and answering it
took an engine-source dive rather than a log read.

**Fix:** one `UE_LOG` on the inline path, at `Log` rather than `Verbose` (it is rare — one line per
tick-unsafe verb that runs inline, not per queue drain), naming that no world is ticking and the game
thread is not draining a named-thread queue. Suggested placement is immediately after the deferral
block's `return`, so only the 35 listed verbs reach it.

**Worth doing before the live-editor verification of that gate rather than after.** Its two regression
tests skip on hosts that cannot enter a nested named-thread pump (`reason=nested-named-thread-pump-not-entered`),
so a log line is currently the only evidence available on such a host that the widened gate behaves as
intended.

## History
- `#1-observability-gap-named-by-the-gate-fix` `OPEN` reporter — Proposed by the agent that fixed
  `B-safepoint-tick-gate-inert-on-simpletickobjects-path`, which supplied the exact diff but correctly
  did not apply it: `Dispatch/RpcDispatcher.cpp` is a central dispatch file outside that ticket's
  ownership. Recorded after the deferral message itself was corrected — it had continued to assert "a
  world is inside `UWorld::Tick`" after the gate grew a third term, and now names whichever term fired.
- `#2-inline-path-now-logs` `IN-REVIEW` developer — "Added a `Log`-level
  `UE_LOG(LogPinWrightSafePoint, ...)` on the inline path in `Dispatch/RpcDispatcher.cpp`, immediately
  after the deferral block's `return` and guarded by a hoisted `bTickUnsafeMethod` local so only the 35
  table entries reach it; the message names both terms of `IsSafeNow()` — no world is inside
  `UWorld::Tick`, and the game thread is not draining a task-graph named-thread queue. Regression test
  `PinWright.core.safe_point.DispatcherLogsInlineTickUnsafeRun` in `Tests/World/TestSafePointGate.cpp`
  marks the `_test.beta` fixture tick-unsafe, dispatches it on the real automation stack and asserts
  exactly one captured `LogPinWrightSafePoint` line carrying the request id and both gate terms. It
  needs no nested named-thread pump, so unlike the two skipping pump tests it runs on this host. Not
  compiled or executed here — the build/test loop is the verification."
