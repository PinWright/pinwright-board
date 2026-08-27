---
id: B-test-invokehandler-bypasses-param-gate
title: "TestUtils.h InvokeHandler skips ValidateHandlerParams, so no test written that way can see an undeclared-parameter defect"
status: OPEN
severity: High
category: bug
tags: [test-gap, test-harness, InvokeHandler, ValidateHandlerParams, dispatcher, undeclared-parameter, guard-inefficacy]
encounters: 1
lastSeen: 2026-08-27
---

# The test helper most handler tests use cannot see a whole class of bug

`Tests/TestUtils.h`'s `InvokeHandler` calls the handler body directly and **skips
`ValidateHandlerParams`** -- the dispatcher's parameter gate. A verb whose handler reads a parameter it
never declared therefore passes every test written with `InvokeHandler`, while every real caller is
rejected with `UNKNOWN_PARAMS` before the handler runs.

That is not hypothetical. `B-niagara-set-parameter-emitter-scope-unreachable` was exactly this: the
whole resolution pipeline for the three emitter-scoped parameter stores already worked, only the
declaration was missing, and the verb's tests were green because they never went through the
dispatcher. Two more instances were then found by sweeping the same namespace
(`niagara.add_parameter`, `niagara.remove_parameter`).

**Fix direction, and it is a judgement call.** The bypass is presumably deliberate -- it lets a test
drive a handler without constructing a full payload. Options: add an
`InvokeHandlerThroughDispatcher` alongside it and steer new tests to that; or run the param gate
inside `InvokeHandler` and let tests opt out explicitly.

Either way the fix is only half the work. The other half is a sweep for verbs whose handler reads a
parameter their `RPC_PARAMS` does not declare -- mechanically checkable, and it would catch the rest
of these before a caller does.

## History
- `#1-found-via-a-dead-parameter` `OPEN` reporter -- Found by the agent fixing
  `B-niagara-set-parameter-emitter-scope-unreachable`, which had to route both of its new tests through
  a real `FRpcDispatcher` because `InvokeHandler` cannot observe this defect. The two sibling verbs it
  found by sweeping every `REGISTER_RPC_HANDLER` param list in `Handlers/Niagara/` are fixed; the
  harness gap that hid them is not.
