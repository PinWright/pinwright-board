---
id: B-registration-summaries-promise-absent-fields
title: "system.run_tests registration summary promises pass/fail counts that only the isolateGroups path returns"
status: OPEN
severity: Medium
rice: [2, 2, 1, 1]
priority: 33
category: bug
tags: [wiki, discoverability, jobs, system, response-schema, false-promise]
costly: 1
---

# system.run_tests summary promises pass/fail counts that the default paths never return

A verb's `REGISTER_RPC_HANDLER` summary becomes its generated wiki line, which is how an agent learns
what a call returns. `system.run_tests` is registered as *"Run UE automation tests and report
pass/fail counts"* (`Source/PinWright/Private/Handlers/System/SystemControlHandler.cpp:892`). Only one
of its three result paths has counts:

- **Exact names (`test` / `tests`):** `MakeRunTestsResult` (`:180-190`) returns `{has_errors,
  requestedTests[], resolvedTests[], missingTests[]}`. No counts.
- **Filter pattern:** `HandleTestsComplete` (`:545-561`) returns `{has_errors}` only; the
  never-started branch (`:532-540`) adds `reason`.
- **`isolateGroups`:** each `groups[]` entry carries `found`, `started`, `succeeded`, `failed`,
  `skipped`, `performed` (`AddGroupResult`, `:786-803`). This is the only path that matches the
  summary.

On the two default paths the call succeeds and the promised counts are absent, so the caller first
suspects its own parsing. `B-system-run-tests-no-completion-signal` (DONE) was written from the
summary and carries the wrong payload shape.

**Workaround:** read the counts from the editor log or the automation report, or pass
`isolateGroups: true`.

**Fix:** reword the summary to the real payload: `has_errors` on every path, plus the
requested/resolved/missing name lists for exact names, and per-group counts only with
`isolateGroups`. Or add pass/fail/skip counts to the name and filter results. Add a test that
asserts the result keys of each path against the summary.

**Acceptance:** the generated `system.run_tests` wiki line names no field that the name and filter
paths do not return. A test fails if the name-path or filter-path result gains or loses a key
without the summary changing.

Related: `E-module-rename-citation-sweep` (where this was found), `B-performance-run-benchmark-measures-nothing`
(same class: a payload shape cited as a contract that the handler does not produce).

## History
- `#1-initial-report` `OPEN` reporter — Three registration summaries name response fields their handler never emits, verified at plugin HEAD `ef8a1f1b`. `system.run_tests` (`SystemControlHandler.cpp:477`) advertises "report pass/fail counts"; `MakeRunTestsResult` (`:145-156`) returns `{has_errors, requestedTests[], resolvedTests[], missingTests[]}` and the filter path (`:541`) returns `{has_errors}` alone. `system.run_ubt` (`:366`) advertises "capture stdout/stderr"; `ProcPollBind.h:37-38` sets `{exit_code}` only, and `SystemControlHandler.cpp:448` comments "polls only the child's exit code (no output pipe)" — summary and comment contradict in one file. `system.inspect.inspect_class` (`EnvironmentHandler.cpp:1632-1633`) understates instead, advertising only "name, full path, and parent class" while the handler emits `functions[]` (`:1681`) and `properties[]` (`:1693`) per `F-inspect-class-members`. Two DONE tickets (`B-system-run-tests-no-completion-signal`, `B-system-run-ubt-no-completion-signal`) were written from the summaries rather than the handlers and carry the wrong payload shape through a tester pass. Severity Medium: a readback that omits an advertised field and forces a source dive is the rubric's soft-blocker band — not High, because no wrong *value* is returned and the call itself is honest; not Low, because this is not a missing doc but an advertised field that does not exist, which sends the caller hunting a bug in its own parsing. Reach neutral: `system.*` job verbs are reached deliberately, not every session.
- `#2-run-ubt-removed` `OPEN` developer - `system.run_ubt` has been deleted from the plugin (registration, `ProcPollBind.h` and `UbtEntryPoint.h` removed; CHANGELOG `## Unreleased`). Item 2 is therefore moot: there is no summary left to correct and no response keys to assert for it. Items 1 (`system.run_tests` "report pass/fail counts", still in its registration) and 3 (`system.inspect.inspect_class` understating) are unaffected, so the ticket stays OPEN with its scope reduced to those two.
- `#3-rephrased` `OPEN` developer — Narrowed to `system.run_tests`. Item 2 (`system.run_ubt`) was removed in `#2`. Item 3 is fixed: the `system.inspect.inspect_class` summary now names inheritanceChain, properties, functions and interfaces (`EnvironmentHandler.cpp:1646`). Item 1 refined: the summary (`SystemControlHandler.cpp:892`) is true only for `isolateGroups`, whose `groups[]` carry found/started/succeeded/failed/skipped (`:786-803`). The name path (`MakeRunTestsResult`, `:180-190`) and the filter path (`:545-561`) return no counts. Refreshed citations and added Workaround and Acceptance. Severity unchanged (Medium).
