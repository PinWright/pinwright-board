---
id: B-editor-run-tests-false-failure-on-green-run
title: "editor_run_tests returns EDITOR_TESTS_FAILED for a run that finished green (3761/3761, exit code 0)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [testing, automation, tooling, misleading-failure, completion-signal, mcp-tool]
---

# `editor_run_tests` reports failure for a fully successful run

The `editor_run_tests` MCP tool returned:

```
EDITOR_TESTS_FAILED: Unreal automation failed, crashed, exited nonzero, or left tests unfinished.
```

for a run that did none of those things.

## Evidence

Run `0d239578699045a18081f8640b284650`, filter `PinWright`:

- `report/index.json`: `succeeded: 3110`, `succeededWithWarnings: 651`, `failed: 0`,
  `notRun: 0`, `tests[]` length **3761**, every entry `state: "Success"`.
- `automation.log`: `Test Started` **3761**, `Result={Success}` **3761**, `Result={Fail}` **0**
  — `started == success + fail`, so the queue drained.
- `automation.log` tail: `**** TEST COMPLETE. EXIT CODE: 0 ****`.
- ZenServer outage probe (`Failed to connect to localhost port 8558`): **0**.

Reproduced on an earlier run in the same session (`5d2fd6a7b63d4696a57d99f222c43b4c`,
filter `PinWright.material`): 149 started, 149 success, 0 fail, exit code 0, same
`EDITOR_TESTS_FAILED`.

## Likely cause (not yet confirmed in source)

Neither log contains the phrase `Automation Test Queue Empty`. `CLAUDE.md` documents that
phrase as the authoritative terminal condition, emitted by
`AutomationCommandline.cpp:122` only from `IsTestingComplete()`, and reached by passing
`-TestExit="Automation Test Queue Empty"`. The wrapper appears to treat the absence of that
marker as "tests unfinished" while the run in fact ended through the
`LogAutomationCommandLine: Shutting down` / `TEST COMPLETE` path. Either the wrapper does not
pass `-TestExit`, or it requires a marker this exit path never emits.

## Why it matters

This is the **misleading-failure** mirror of the misleading-success class in
`docs/rpc-design.md` §1/§5b. A caller who trusts the tool's verdict concludes a green suite
is red, and the natural response — re-running, or hunting a nonexistent regression — costs a
full suite run each time (this one took 1052 s). It also trains callers to ignore the tool's
verdict and read the log themselves, which defeats the tool.

## Suggested fix

Decide completion from the evidence the run actually produces: `report/index.json`'s
`failed` / `notRun` counts plus the process exit code, rather than from a log phrase the exit
path may not emit. If the marker is wanted, pass `-TestExit="Automation Test Queue Empty"`
explicitly so the condition it checks is the condition it created.

## History
- `#1-false-failure-on-green-run` `OPEN` reporter — `editor_run_tests` returned `EDITOR_TESTS_FAILED` for run `0d239578699045a18081f8640b284650` (filter `PinWright`), which finished with `report/index.json` reporting 3761 tests all `state:"Success"`, `failed:0`, `notRun:0`, the log showing 3761 `Test Started` / 3761 `Result={Success}` / 0 `Result={Fail}`, `**** TEST COMPLETE. EXIT CODE: 0 ****`, and a ZenServer probe of 0. Reproduced on `5d2fd6a7b63d4696a57d99f222c43b4c` (filter `PinWright.material`, 149/149 green). Neither log contains `Automation Test Queue Empty`, which `CLAUDE.md` names as the terminal marker, so the wrapper is likely gating on a phrase this exit path does not emit. Found while verifying `B-compile-mgir-landscape-consumers-stale`; the suite there is genuinely green and had to be confirmed by reading the report JSON by hand.
- `#2-both-halves-fixed` `IN-REVIEW` developer — Fixed in `3151e26a`, and `#1`'s "likely cause (not yet confirmed in source)" turned out to be **two independent defects**, either of which alone produces the reported symptom. (a) **The report path never worked.** UE writes the automation report's `index.json` with a UTF-8 BOM, so `json.load` raised on every real report and each one was classified invalid — which is why the verdict fell through to the log fallback in the first place, and why the `report/index.json` evidence quoted in `#1` (3761 entries, all `state:"Success"`, `failed:0`, `notRun:0`) never reached the verdict at all. (b) **The log fallback searched for a marker the launch had not asked the engine to emit.** `#1` guessed correctly: the wrapper shipped `; Quit`, which is simply the next command queued on the same `-ExecCmds` list and is not a completion condition — both of its failure modes are on record for this suite, firing EARLY (a false green that released the editor after 2012 of 3658 tests) and never firing at all. The launch now passes `-TestExit="Automation Test Queue Empty"`, so the condition the verdict checks is the condition the launch created. Worth recording as the story rather than a footnote: the predecessor `f0ab5a74` prescribed exactly that flag a full day earlier — **in `CLAUDE.md` only** — while the wrapper went on shipping `;Quit`. Documenting the right invocation next to a tool that does not use it is how a fix gets believed without being applied.
