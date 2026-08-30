---
id: F-controlrig-test-coverage
title: "controlrig.* RPC handler-envelope tests"
status: IN-REVIEW
severity: Low
category: feature
tags: [tests, controlrig]
---

# controlrig.* RPC handler-envelope tests

The two registered handlers in `Source/PinWright/Private/Handlers/ControlRig/` —
`controlrig.compile_crir` (`CRIRCompileHandler.cpp`) and
`controlrig.decompile_crir` (`CRIRDecompileHandler.cpp`) — have no automation
test that invokes them through the dispatcher. The underlying
`FCRIRCompiler` / `FCRIRDecompiler` engine is heavily covered (13 tests under
`Private/Tests/Assets/TestCRIR*.cpp`, plus the `F-crir-coverage-test-matrix`
node-kind matrix), so the round-trip logic is not at risk. What is untested is
the thin RPC-envelope layer: param extraction (`text`, `context`, `mode`,
`runLayout`, `save`), the `replace` / `extend` mode parser (`TryParseCRIRCompileMode`),
error code mapping (`INVALID_PARAMS` on bad mode, propagation of
`CompileResult.ErrorCode`), and the success-payload shape (`mode`, `assetPath`,
`blocksCompiled`, `nodesCreated`, `warnings`).

Note: original triage flagged this namespace as a P0 "fully untested namespace."
Reframed to Low after confirming compiler/decompiler coverage is comprehensive
and only the handler shell is unexercised.

## Fix

Add `IMPLEMENT_SIMPLE_AUTOMATION_TEST` cases under `Private/Tests/Handlers/`
(or `Private/Tests/Assets/`) that drive `FRpcDispatcher` directly with
`controlrig.compile_crir` / `controlrig.decompile_crir` JSON payloads:

- Bad `mode` value → expect `INVALID_PARAMS` with the "Unknown CRIR compile mode" message.
- Missing required `text` / `context` → expect the standard `RequireString` failure.
- Valid `mode=replace` and `mode=extend` round-trip → success payload shape assertions.
- `save=true` path → assert asset marked dirty (or at minimum that the flag is consumed).

Reuse the synthetic Control Rig BP fixtures from `Tests/Assets/CRIRTestHelpers.h`.

## History
- `#1-initial-scope` `OPEN` reporter — Audit found `controlrig.*` is the only namespace whose RPC handler files have no test invoking them via the dispatcher. Underlying compiler engine is well-covered; gap is limited to the param/error/payload shell. Filing as Low because the risk surface is small and the handlers are short delegators, not P0 as the initial triage suggested.
- `#2-handler-envelope-tests` `IN-REVIEW` developer — Added `Private/Tests/Assets/TestCRIRHandlers.cpp` with 7 `IMPLEMENT_SIMPLE_AUTOMATION_TEST` cases driving `controlrig.compile_crir` / `controlrig.decompile_crir` through `InvokeHandlerWithCapture`: bad mode → `INVALID_PARAMS` + "Unknown CRIR compile mode", missing `text`/`context` → error, valid `replace`/`extend` round-trip → success-payload shape (`mode`/`assetPath`/`blocksCompiled`/`nodesCreated`/`warnings`), decompile valid asset → `assetPath`/`text`/`warnings`, decompile missing `assetPath` → error. Reuses `McpCreateControlRigBlueprint` + `GetFirstModelController` fixtures; valid-compile tests decompile a wired source BP to obtain valid CRIR text. No production code changed.
- `#3-returned-extend-test-fails` `OPEN` tester — Ran the 7 tests live via `system.run_tests` (job j_20260605T105414, status `failed`/`TESTS_FAILED`). 6 pass; `controlrig.compile_crir.ValidModeExtend` FAILS. Root cause: the `extend`-mode target BP already contains the `BeginExecution`/Forwards Solve event (added by `MakeWiredControlRigBP`), and the decompiled source CRIR re-adds it, raising `LogScript: Error: Event Forwards Solve already exists in the graph`, which the automation framework treats as a test failure. `ValidModeReplace` passes because replace clears the target graph first. The fix must use an empty (or eventless) target for the extend case, or assert/expect the existing event rather than duplicating it. Test: `system.run_tests tests=[...the 7 names...]`; PDS.log line 4026 `Result={Fail} Name={ValidModeExtend}`, line 4028 captures the script error.
- `#4-preflight-extend-event` `IN-REVIEW` developer — Updated `CRIRCompiler.cpp` so extend-mode unit compilation preflights existing singleton event nodes before calling `AddUnitNodeFromStructPath`, avoiding Unreal's duplicate Forwards Solve event path while keeping the post-add fallback defensive. Tightened `FCRIRCompileValidModeExtendTest` in `TestCRIRHandlers.cpp` to keep the wired target and assert the success payload includes `assetPath`, `blocksCompiled`, `nodesCreated`, and `warnings` without requiring extend to create new nodes.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
