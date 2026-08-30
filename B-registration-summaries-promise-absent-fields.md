---
id: B-registration-summaries-promise-absent-fields
title: "Three registration summaries advertise response fields the handler never returns"
status: OPEN
severity: Medium
category: bug
tags: [wiki, discoverability, jobs, system, response-schema, false-promise]
---

# Three registration summaries advertise response fields the handler never returns

A verb's `REGISTER_RPC_HANDLER` summary is the generated wiki, and the wiki is an agent's only
discovery surface for what a call returns. Three summaries name fields the handler does not
produce. The call then succeeds and the promised key is simply absent, so the caller looks for the
bug in its own parsing before it thinks to read the handler.

Found while sweeping citations for `E-module-rename-citation-sweep`; all three verified at plugin
HEAD `ef8a1f1b`.

1. **`system.run_tests` — "report pass/fail counts".** Registration
   `Source/PinWright/Private/Handlers/System/SystemControlHandler.cpp:477`. The completion payload
   is `MakeRunTestsResult` (`:145-156`): `{has_errors, requestedTests[], resolvedTests[],
   missingTests[]}`. There is no `passed`, `failed` or `skipped` anywhere. The filter path (`:541`)
   is thinner still — `{has_errors}` alone.
2. **`system.run_ubt` — "capture stdout/stderr".** Registration `:366`.
   `Source/PinWright/Private/Handlers/BuildTools/ProcPollBind.h:37-38` sets exactly one field,
   `{exit_code}`. The plugin knows: `SystemControlHandler.cpp:448` comments that it *"polls only
   the child's exit code (no output pipe)"* — the summary and the code comment contradict each
   other in the same file.
3. **`system.inspect.inspect_class` — "name, full path, and parent class".** Registration
   `Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp:1632-1633`. This one errs
   the other way: the handler *also* emits `functions[]` (`:1681`) and `properties[]` (`:1693`),
   which is what `F-inspect-class-members` (DONE) delivered — so the summary understates and a
   caller has no reason to ask for the thing that already works.

The board has already paid for two of these once. `B-system-run-tests-no-completion-signal` and
`B-system-run-ubt-no-completion-signal` are both `DONE`, and both describe the promised payload
rather than the real one — the ticket text was written from the summary, and the summary was never
checked against the handler. So the wrong shape propagated from the registration into the board and
sat there through a tester pass.

**Fix:** correct the three summaries to the payload the handler actually returns, and add a
regression test that asserts the response keys of each. The general form is worth considering
separately: nothing gates a summary's claims against the handler's emitted keys, and a summary is
the one part of a handler that no test exercises.

**Related:** `E-module-rename-citation-sweep` (where these were found and what else it turned up).
Same class as `B-performance-run-benchmark-measures-nothing`, which caught a payload shape invented
in a history entry and cited afterwards as a contract — here the invention is one level upstream,
in the shipped registration.

## History
- `#1-initial-report` `OPEN` reporter — Three registration summaries name response fields their handler never emits, verified at plugin HEAD `ef8a1f1b`. `system.run_tests` (`SystemControlHandler.cpp:477`) advertises "report pass/fail counts"; `MakeRunTestsResult` (`:145-156`) returns `{has_errors, requestedTests[], resolvedTests[], missingTests[]}` and the filter path (`:541`) returns `{has_errors}` alone. `system.run_ubt` (`:366`) advertises "capture stdout/stderr"; `ProcPollBind.h:37-38` sets `{exit_code}` only, and `SystemControlHandler.cpp:448` comments "polls only the child's exit code (no output pipe)" — summary and comment contradict in one file. `system.inspect.inspect_class` (`EnvironmentHandler.cpp:1632-1633`) understates instead, advertising only "name, full path, and parent class" while the handler emits `functions[]` (`:1681`) and `properties[]` (`:1693`) per `F-inspect-class-members`. Two DONE tickets (`B-system-run-tests-no-completion-signal`, `B-system-run-ubt-no-completion-signal`) were written from the summaries rather than the handlers and carry the wrong payload shape through a tester pass. Severity Medium: a readback that omits an advertised field and forces a source dive is the rubric's soft-blocker band — not High, because no wrong *value* is returned and the call itself is honest; not Low, because this is not a missing doc but an advertised field that does not exist, which sends the caller hunting a bug in its own parsing. Reach neutral: `system.*` job verbs are reached deliberately, not every session.
