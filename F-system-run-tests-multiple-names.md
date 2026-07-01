---
id: F-system-run-tests-multiple-names
title: "system.run_tests should accept multiple exact test names"
status: DONE
severity: Medium
category: feature
tags: [system, automation, tests, jobs]
---

# system.run_tests should accept multiple exact test names

`system.run_tests` currently accepts `filter` or a single `test` string, then shells the selection through `GEngine->Exec("Automation RunTests ...")`. That makes the RPC contract string-shaped even though callers often know the exact tests they want to run. UE's console command does support multiple selectors by joining them with `+`, but routing a structured RPC array through console parsing preserves avoidable ambiguity around exact-name selection, validation, quoting, and no-match behavior.

UE 5.6 exposes the needed `IAutomationControllerManager` APIs publicly: `RequestAvailableWorkers`, `RequestTests`, `OnTestsRefreshed`, `SetFilter`, `GetFilteredTestNames`, `SetEnabledTests`, `RunTests`, `OnTestsComplete`, and `ReportsHaveErrors`. The cleaner implementation is a small controller-backed runner for `system.run_tests`, not an array-to-console-string adapter.

**Workaround:** Call `system.run_tests` once with a broad `filter`, or pass a manually joined selector string through the existing single `test`/`filter` path if the caller knows UE's `Automation RunTests A+B` syntax.

**Proposal:** Add an optional `tests` array parameter for exact test names while keeping existing `test` and `filter` compatibility. For exact-name input, drive `IAutomationControllerManager` directly: request workers/tests, wait for `OnTestsRefreshed`, resolve the requested names against the discovered test list, call `SetEnabledTests(ResolvedNames)`, then `RunTests()`. Return the canonical job ticket from `Ctx.StartJob()` and include `requestedTests`, `resolvedTests`, and `missingTests` in kickoff or completion payloads. If no requested tests resolve, complete/fail the job deterministically instead of leaving it running.

## Plan

1. Extract `system.run_tests` selection into a small helper owned by the system handler layer, preserving `filter` and single `test` behavior.
2. Add `tests: string[]` schema support. Reject requests that combine `filter` with `test`/`tests`; accept `test` as a single-entry exact-name compatibility alias.
3. Replace the exact-name path's `GEngine->Exec("Automation RunTests ...")` call with direct `IAutomationControllerManager` orchestration: initialize controller, request workers/tests, bind one-shot `OnTestsRefreshed`, resolve exact names, set enabled tests, run, then complete through `OnTestsComplete`.
4. Keep the broad `filter` path either on the existing console command initially or move it through the same direct runner using UE's `AutomationFilterCollection`, depending on implementation risk. Do not broaden scope into unrelated job-registry changes.
5. Add focused automation tests for payload parsing and selection behavior: legacy `test`, new `tests` array, `filter` conflict rejection, empty `tests`, duplicate names, and no-match deterministic failure.
6. Update the hand-written system wiki overlay and generated schema source only; do not hand-edit `docs/rpc-method-reference.generated.md`.

## History
- `#1-structured-test-selection` `OPEN` reporter — User asked whether `Automation RunTests <name>` should be used at all when extending `system.run_tests` to accept multiple test names. Source research showed the console command is a wrapper over public `IAutomationControllerManager` APIs, so the task is to add structured `tests: string[]` support and prefer direct controller orchestration for exact-name runs.
- `#2-exact-name-runner` `IN-REVIEW` developer — Added `tests` array schema support, validation for `filter` conflicts and malformed arrays, and a direct `IAutomationControllerManager` exact-name runner that resolves requested names before calling `SetEnabledTests`/`RunTests`. Legacy run-all/filter path still uses the existing console command. Added focused handler contract tests and documented the new selection modes in `docs/wiki/system.md`.
- `#3-verified-tests-array` `DONE` tester — Verified live without running real tests: `system.run_tests` help exposes `tests` as an array selection mode, and `system.run_tests tests:["Definitely.NoTestMatchesThis_xyzzy_A","Definitely.NoTestMatchesThis_xyzzy_B"]` returned ticket `j_20260430T142040_a2474ff2` with `selectionMode:"tests"` and both requested names; `system.job_status` then failed deterministically with `NO_TESTS_MATCHED`, `resolvedTests:[]`, and both names in `missingTests`.
