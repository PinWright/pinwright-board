---
id: B-test-invokehandler-bypasses-param-gate
title: "TestUtils.h InvokeHandler skips ValidateHandlerParams, so no test written that way can see an undeclared-parameter defect"
status: IN-REVIEW
severity: High
category: bug
tags: [test-gap, test-harness, InvokeHandler, ValidateHandlerParams, dispatcher, undeclared-parameter, guard-inefficacy]
encounters: 2
lastSeen: 2026-08-28
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
- `#2-guard-test-plus-sweep` `IN-REVIEW` developer -- Kept `InvokeHandler`'s behaviour (changing it would touch ~1000 call sites and would not find this defect class anyway) and closed the gap with a registry-wide guard instead. Added `Tests/Infra/TestDeclaredParamCoverage.cpp` (`PinWright.infra.declared_params.HandlersOnlyReadDeclaredParams`): scans every `REGISTER_RPC_HANDLER` body off disk, collects the literal keys it passes to `Ctx.Get*`/`Ctx.Require*`, compares them against the live registration's accepted names (Name + Aliases + TypedAliases), fails on any pair outside a recorded 66-entry baseline and warns on a baseline entry that stops reproducing. Added `ParamSpecTestHelpers::CollectAcceptedParamNames` / `IsParamAccepted` (the alias-aware declaration check `FindParamSpec` could not answer) and documented the bypass on `Tests/TestUtils.h`'s `InvokeHandler`, pointing at that helper and at `DispatcherTestHelpers`. The offline sweep found 66 direct-read pairs across 36 verbs plus ~34 more read through shared `FHandlerContext&` helpers; the list is in the fix report, unfixed by design.

- `#3-not-decidable-behaviourally` `IN-REVIEW` verifier — 2026-08-28. **Left IN-REVIEW on purpose:
  `#2` changes nothing a live caller can observe, so there is no repro to run.** What I did measure
  against the running editor at `b79ba53e` (UE 5.8):
  - **The dispatcher gate is live and real.** `asset.exists {assetPath, bogusParamXyz:1}` →
    `UNKNOWN_PARAMS: Unknown parameter(s) for 'asset.exists': [bogusParamXyz]. Valid parameters:
    [assetPath, path]`.
  - **The defect class this ticket is about is still present in shipped verbs, exactly as `#2` says
    it left it.** Three of the 66 baseline pairs, in three namespaces, refused at the gate before
    the handler runs: `actor.describe {field:"transform"}` → `UNKNOWN_PARAMS ... [field]`;
    `actor.describe {include_components:true}` → `UNKNOWN_PARAMS ... [include_components]`;
    `niagara.graph.get {emitterName:"Bogus"}` → `UNKNOWN_PARAMS ... Valid parameters: [assetPath,
    emitter, scriptUsage]`. Differential control on the same verb: `actor.describe
    {includeComponents:true}` clears the gate and reaches the body (`ACTOR_NOT_FOUND`). So the
    handlers really do honour keys the declaration refuses, and `#2` fixed none of them — by design.
  - **`InvokeHandler` is unchanged.** `Tests/TestUtils.h:205-218` at HEAD still builds
    `FHandlerContext::MakeTestContext` and calls `Reg.Func(Ctx)` with no `ValidateHandlerParams`;
    the change there is a comment block (`:193-204`) pointing at `ParamSpecTestHelpers::IsParamAccepted`
    and `DispatcherTestHelpers`. The ticket's literal defect statement therefore still holds and
    always will — `#2` substituted a different mechanism rather than closing it.
  - **The guard exists and is soundly built** — `Tests/Infra/TestDeclaredParamCoverage.cpp` (25.7 KB),
    66 `KnownUndeclaredReads` entries (counted), a vacuity guard that fails unless the source scan
    recovers at least half the live registration count, a stale-baseline report so the list cannot
    rot into a permanent exemption, and an explicit marked skip (`plugin-source-tree-absent`) rather
    than a silent pass on a binary-only install.
  **Why this is not decidable here:** the fix is a test, and the only way to know whether it fires is
  to run `PinWright.infra.declared_params.HandlersOnlyReadDeclaredParams`. I was scoped to
  behavioural repros and barred from the automation suite, so I have no measured evidence either way
  and will not manufacture a verdict. To close it, someone should run that one test id in a scoped
  editor run and, separately, prove it is differential: introduce a throwaway undeclared `Ctx.GetString`
  read in one handler and confirm the test fails naming that pair, then revert.
  **Two things `#2` did not cover, found by reading `TestUtils.h` at HEAD:**
  (a) `InvokeHandlerWithCapture` (`:222-237`) has the identical bypass and did not get the doc note
  — the same trap, one line lower, and it is the variant tests reach for when they assert a response;
  (b) `#2`'s own report records ~34 further undeclared reads that go through shared
  `FHandlerContext&` helpers, which the literal-key regex at `:241-244` cannot see — a known blind
  spot in the guard, unrecorded in the baseline and so invisible to the stale-entry reporter too.
