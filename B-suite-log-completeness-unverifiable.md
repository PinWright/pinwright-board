---
id: B-suite-log-completeness-unverifiable
title: "A killed automation suite produces a log that greps clean — zero failures, no crash, no completion marker — and is indistinguishable from a green run; two such logs were quoted today as evidence of green suites, and one widely-quoted suite figure has no log on disk at all"
status: IN-REVIEW
severity: High
category: bug
tags: [testing, automation, suite, completion-signal, false-green, verification, provenance, tooling]
---

# A killed run and a green run grep identically

`rg "Result=\{Fail\}"` returning nothing is **not** a pass. A commandlet killed mid-queue produces a
log with zero `Result={Fail}`, no `Fatal`, no crash, no assertion — and no completion marker either.
Nothing in the usual check distinguishes it from a suite that drained.

## The killed logs, named

Full inventory of the 13 automation logs on this host was taken; ZenServer probe is 0 in all of
them, so no run is discountable on that basis.

| Log | Started | Success | Fail | `Automation Test Queue Empty <N> tests performed` |
|---|---:|---:|---:|---|
| `Saved/Logs/pw_suite_clb.log` | 610 | 609 | **0** | **absent** |
| `Saved/Logs/pw_suite_full2.log` | 762 | 756 | 5 | **absent** |
| `Saved/Logs/pw_suite_ortho.log` | 3778 | 3772 | 6 | present, `:39655` |
| `Saved/Logs/pw_suite_extendguard.log` | 3797 | 3797 | 0 | present, `:39745` |

`pw_suite_clb.log` is the pure case: **610 started, 609 success, 0 fail** — `started != success + fail`,
one test still in flight — spanning `[2026.08.16-07.51.14:606]` to `[2026.08.16-07.54.39:297]` UTC
(12:51-12:54 local; logs are UTC+0, this machine is UTC+5). The file simply stops mid-test:

```
[2026.08.16-07.54.38:742][511]LogAutomationController: Display: Test Started. Name={CreatesSignatureGraphAndProperty} ...
[2026.08.16-07.54.39:297][511]LogDatasmithContent: Do not use the UDatasmithStaticMeshCADImportData ...
```

No `TEST COMPLETE`, no `LogExit`, no `Engine exit requested`, no crash. **It greps clean.** Both it
and `pw_suite_full2.log` were quoted today as evidence of green suites; both were killed commandlets.

## Do not grep the bare phrase

`-TestExit="Automation Test Queue Empty"` is echoed into **every** log's command line at startup
(`LogCsvProfiler: ... commandline=` and `LogInit: Command Line:`), so a bare
`rg -c "Automation Test Queue Empty"` returns 2 in a killed log and 4 in a drained one — a
non-zero count that reads as success. Only the tail form counts:
`Automation Test Queue Empty <N> tests performed`, emitted from `AutomationCommandline.cpp:122`
solely via `IsTestingComplete()`.

## The sharpest evidence: a quoted figure with no log behind it

Commits `88f6ee4f` and `052977bd` both close with **"Suite 3795/3795/0"**. That figure was carried
into three board tickets as the verification rung.

**No log on this host contains `3795 tests performed`.** Every terminal marker present on disk is:
151, 3, 3713, 3749, 3758 (×2), 3760, 3778, 3797, 4 (×2), 80. There is no automation log written in
the 14:19-14:20 local window the commits were authored in. The number may well be true — but it
cannot be re-derived, which under the board's own standard makes it worth little.

This is the failure mode the ticket is about, one level up: **when completeness is unverifiable,
numbers stop being measurements and become recollections**, and nothing downstream can tell which it
is holding.

## The naive grep confirmed a run that never completed — first-hand, 2026-08-29

The orchestrator of this wave ran the suite, grepped the log for the completion marker, and got a
match. The match was the **command-line echo of `-TestExit="Automation Test Queue Empty"`** in the
log preamble. The run had been truncated at **4,312 of 4,625 tests** with **zero** real TestExit
markers. So the standard check confirms completion on a run that never completed, in the hands of
the person who wrote the rule, on the same day the rule was being enforced elsewhere.

## The defect has already produced a second ticket built on misattributed evidence

`B-suite-host-gc-crash-in-combined-group-run` rests on three "crashes". A forensic pass over the
archived runs found **two of the three were not crashes**: `batch4` (stopped at 4009) and `batch5`
(stopped at 2225) each end on a complete newline-terminated line with no `EXCEPTION_ACCESS_VIOLATION`,
no `Fatal`, no `RequestExitWithStatus`, and **neither produced a crash report** — the newest
non-ensure report in `Saved/Crashes` predates both runs, and the three reports inside their windows
are all `IsEnsure=true`. Both stop on the identical two-line sequence inside
`ObjectTools::ForceDeleteObjects`.

Re-verified independently in `unreal-fpv-new` while implementing the fix: `batch4` = 4009 started /
4007 success / 1 fail, `batch5` = 2225 / 2224 / 0, both with the bare phrase present **twice** (the
echo) and the counted tail **absent**; `batch6` = 4576 / 4576 / 0 with the bare phrase 4× and the
tail once. Crash-report sweep of the windows: every report `IsEnsure=true`, newest non-ensure report
`2026-08-27 20:39` (Assert), before either truncation ended.

That is the cost of this defect stated concretely: when completeness is unverifiable, a truncation
is read as a crash, a ticket is filed on it, and a severity argument is built on top.

## Proposed fix — with a correction to the obvious wording

The tempting rule is "require `Automation Test Queue Empty` and treat its absence as
did-not-complete". **That rule is wrong as stated** and would re-create the defect already filed as
`B-editor-run-tests-false-failure-on-green-run`: the `editor_run_tests` wrapper's own runs exit
through a different, equally real terminal path. Both logs cited in that ticket
(`Saved/PinWright/test-runs/0d239578699045a18081f8640b284650/automation.log`, 3761/3761/0, and
`.../5d2fd6a7b63d4696a57d99f222c43b4c/automation.log`, 149/149/0) lack the queue-empty marker while
carrying:

```
LogAutomationWorker: Received StopTestSession from ...
LogAutomationCommandLine: Shutting down. GIsCriticalError=0
LogAutomationCommandLine: Display: **** TEST COMPLETE. EXIT CODE: 0 ****
```

plus a sibling `report/index.json`. Requiring only the queue-empty marker would fail those green
runs — the exact misclassification that ticket complains about.

**Require *a* terminal marker, not *the* terminal marker:**

- `Automation Test Queue Empty <N> tests performed` (the `-TestExit` path), **or**
- `TEST COMPLETE. EXIT CODE: 0` together with `report/index.json` showing `notRun: 0`,

and treat the absence of **both** as `DID_NOT_COMPLETE` — never as "no failures". Keep the three
existing conditions alongside it: `started == success + fail`, the started count reconciles against
the expected total, and the ZenServer probe is 0.

Note the rule already exists in prose. `Plugins/PinWright/CLAUDE.md` states it correctly ("a run that
ends early greps clean too", plus the three conditions), and `pw_suite_clb.log` fails all three of
them. **The gap is enforcement, not documentation** — the line immediately after says a run
*"**should** end with the automation test-complete marker"*, and "should" is what makes it
unverifiable. It needs to be machine-checked by the same code that renders the verdict, so a human
or an agent cannot skip it under time pressure.

## Related

- `B-editor-run-tests-false-failure-on-green-run` — the mirror defect, and the reason the naive
  wording of the fix is unsafe. These two must be fixed together or they will fight.
- `H-suite-ensure-crash-signal-ambiguous` (harness) — adjacent: what counts as a crash signal in a
  suite log.

## History
- `#1-killed-runs-grep-clean` `OPEN` reporter — A commandlet killed mid-queue produces an automation log with zero `Result={Fail}`, no crash, no `Fatal`, no assertion and no completion marker, and nothing in the usual check distinguishes it from a drained suite. Named instance, from a full 13-log inventory of this host (ZenServer probe 0 in every one): **`Saved/Logs/pw_suite_clb.log` — 610 started, 609 success, 0 fail**, so `started != success + fail` with one test still in flight, spanning `[2026.08.16-07.51.14:606]` to `[2026.08.16-07.54.39:297]` UTC and simply stopping mid-test after a `Test Started` line with no `TEST COMPLETE`, no `LogExit` and no `Engine exit requested`. It greps clean. `Saved/Logs/pw_suite_full2.log` (762/756/5) is the same shape; both were quoted today as evidence of green suites and both were killed. Counting trap to avoid in any fix: `-TestExit="Automation Test Queue Empty"` is echoed into every log's command line at startup, so a bare grep for the phrase returns 2 in a killed log and 4 in a drained one — only the `<N> tests performed` tail form, emitted from `AutomationCommandline.cpp:122` via `IsTestingComplete()`, is evidence.
- `#2-a-quoted-figure-with-no-log-behind-it` `OPEN` reporter — The consequence, measured. Commits `88f6ee4f` and `052977bd` both close with **"Suite 3795/3795/0"**, and that figure was carried into three board tickets as their verification rung. **No log on this host contains `3795 tests performed`.** The complete set of terminal markers present on disk is 151, 3, 3713, 3749, 3758 (twice), 3760, 3778, 3797, 4 (twice) and 80, and no automation log was written in the 14:19-14:20 local window those commits were authored in. The figure may be true; it cannot be re-derived. That is the defect one level up — when completeness is unverifiable, suite numbers stop being measurements and become recollections, and nothing downstream can tell which it is holding.
- `#3-the-obvious-fix-wording-is-wrong` `OPEN` reporter — Recording this before anyone implements it, because the natural rule breaks a sibling ticket. "Require `Automation Test Queue Empty` and treat its absence as did-not-complete" would misclassify the `editor_run_tests` exit path: both logs cited in `B-editor-run-tests-false-failure-on-green-run` (`Saved/PinWright/test-runs/0d239578699045a18081f8640b284650/automation.log` at 3761/3761/0 and `.../5d2fd6a7b63d4696a57d99f222c43b4c/automation.log` at 149/149/0) lack the queue-empty marker while carrying `LogAutomationWorker: Received StopTestSession`, `Shutting down. GIsCriticalError=0`, `**** TEST COMPLETE. EXIT CODE: 0 ****` and a sibling `report/index.json` — they are genuinely complete by a different route. Require **a** terminal marker rather than **the** terminal marker: the queue-empty tail, OR `TEST COMPLETE. EXIT CODE: 0` plus `report/index.json` with `notRun: 0`; absence of both is `DID_NOT_COMPLETE` and never "no failures". Keep `started == success + fail`, the reconciled expected total, and the ZenServer probe alongside it. The rule is already stated correctly in prose in `Plugins/PinWright/CLAUDE.md` and `pw_suite_clb.log` fails all three of its conditions, so **the gap is enforcement, not documentation** — the following line says a run *should* end with the marker, and "should" is what leaves it unverifiable. It has to be checked by the same code that renders the verdict, so the check cannot be skipped under time pressure. Note for whoever picks this up: a guard of exactly this shape was being written in the working tree at filing time and was not yet committed — reconcile with it rather than starting over.
- `#4-additional-current-gates-still-green` `OPEN` reporter — Additional evidence: **Adversarial review A — current enforcement still admits false-green suite evidence.** Actuality: CONFIRMED CURRENT. Framing: accurate, and High remains justified because the proxy and CI normal paths can accept a partial/killed run as green; docs state the required checks but do not enforce them. Proposed fix: INCOMPLETE, the any-terminal-marker rule correctly preserves the sibling `TEST COMPLETE` path but must be implemented once and consumed by the proxy, CI, and both workflow recipes, with machine-checked expected-total/started reconciliation and ZenServer evidence. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:778`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:810`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:1763`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\.github\workflows\ci.yml:95`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\.github\workflows\ci.yml:97`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\.github\workflows\ci.yml:113`, and `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\tests\test_mcp_proxy_editor_start.py:1305` / `:1791` (no-marker report success is accepted and log parsing is suppressed). Runtime: NOT VERIFIED; narrow pure-Python evidence tests pass 10/10, and a valid report with 609 successes, zero unfinished/failures, no terminal log evaluates green, but no Unreal suite was launched. Recommendation: KEEP; centralize the completion verdict, update every launcher/gate and add no-marker regression fixtures, then run a bounded real suite and archive report, log, command, and exit evidence.
- `#5-additional-bom-and-crosscheck` `OPEN` reporter — Additional evidence: **Adversarial review B — A's current-defect verdict survives, but its proposed gate needs a report-integrity cross-check.** Actuality: CONFIRMED CURRENT. Framing: accurate; High remains justified. The proxy launches `Automation RunTests …; Quit` (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:1763`), requires `queueEmpty` only for log evidence (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:778-818`), and skips log parsing whenever a report is valid (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:1811-1817`). CI resets the child exit code and gates only report counts (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\.github\workflows\ci.yml:94-114`); the test-loop recipe still teaches `;Quit` and bare failure greps (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\.polyskill\skills\mcp-test-loop\SKILL.md:169-200`). A further sibling interaction is live: real `index.json` artifacts begin with UTF-8 BOM, while `parse_automation_report` uses `encoding="utf-8"` (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:685`), so the current report path rejects them and can misclassify the valid `TEST COMPLETE` route. UE confirms the routes are distinct: queue-empty is emitted only from `IsTestingComplete()` (`C:\UE_5.8\Engine\Source\Developer\AutomationController\Private\AutomationCommandline.cpp:117-125`), while `TEST COMPLETE` is emitted by the queued Quit path (`C:\UE_5.8\Engine\Source\Developer\AutomationController\Private\AutomationCommandline.cpp:478-504`). Proposed fix: INCOMPLETE, centralization is systemic but any marker plus `notRun:0` is insufficient without BOM-safe parsing, expected-total/started reconciliation, and ZenServer evidence. Evidence: the killed log still ends mid-test at `X:\src\unreal\EAContentExamples58\Saved\Logs\pw_suite_clb.log:12107-12113` (610 started, 609 success, no terminal marker); current pure-Python evidence tests pass 10/10 plus the report/log suppression test, but they encode the false-green acceptance (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\tests\test_mcp_proxy_editor_start.py:1305-1312` / `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\tests\test_mcp_proxy_editor_start.py:1791-1809`). Runtime: NOT VERIFIED this turn; only existing logs/artifacts were inspected. Recommendation: KEEP; fix this with `B-editor-run-tests-false-failure-on-green-run`, add BOM/no-marker/partial-report fixtures, then run a bounded real suite with archived report, log, command, and exit evidence.
- `#6-completeness-is-now-machine-checked-and-crash-is-a-separate-verdict` `IN-REVIEW` developer — Enforced the rule in the code that renders the verdict, in `unreal-fpv-new`. **Marker.** `parse_automation_log` no longer regexes the whole file: `_find_terminal_marker` scans by LINE and requires three properties — shape (`<N> tests performed`, which an echo cannot carry), **source** (a candidate on a line matching `Command Line:` / `commandline=` / `-TestExit` / `-ExecCmds` is the launch quoting itself and is discarded outright, so even a command line carrying the full tail form cannot satisfy the check), and **position** (the marker must follow the last `Test Completed. Result=` line, since both engine emissions come after every test result). The `TEST COMPLETE. EXIT CODE` route is preserved, so `B-editor-run-tests-false-failure-on-green-run` is not re-created. **Crash vs truncation.** New `STATE_CRASHED` / `EDITOR_TESTS_CRASHED`, decided only on positive evidence: a fatal/assert banner, or a **non-ensure** crash report in `Saved/Crashes` inside the run window. `scan_crash_reports` finds the directory by walking up from the log to `Saved/`, anchors the window on the log file's own mtime plus the run duration parsed from the log's first/last timestamps (a difference, so no UTC-vs-local conversion can shift it) with 300 s slack, reads only the head of each `CrashContext.runtime-xml`, and counts `IsEnsure=true` separately. A fatal banner no longer reads `COMPLETED_WITH_FAILURES` (no test failed — the process died), and `DID_NOT_COMPLETE` now states in its reason how many non-ensure and ensure-only reports were in the window, or that the scan was skipped. An ensure alone never sets `fatal`. **Provenance.** Every verdict prints `provenance: <abs log path>:<line>  <kind> marker: <text>`, or, on a truncated run, `NO TERMINAL MARKER ... (the bare phrase appears Nx, all command-line echo)`; `crashReports: N non-ensure, N ensure-only in <dir>` prints on green runs too. **Default path.** The crash scan is on by default (`--no-crash-scan` says "not checked" rather than "none"); `mcp-test-loop`'s Phase 3b now runs the checker BEFORE any grep, and its crash-marker grep no longer treats `EnsureFailed` as a crash — that grep is what mis-filed `batch4`/`batch5`. **Self-test.** New `CompletenessSelfTest` in `Content/Python/tests/test_suite_verdict_gates.py`: five synthetic fixtures (truncated / truncated-carrying-the-argument-echo / crashed / completed-with-failures / completed-clean) asserted against each other — the echo one is asserted to classify IDENTICALLY to the plain truncation, which is the point, plus a fixture-integrity test proving the naive grep IS satisfied by it while the checker is not — with crash-route tests over a synthetic `Saved/Crashes` (ensure-only stays a truncation, a non-ensure report becomes CRASHED naming it, a 48-h-old report is out of window, discovery works from a `test-runs/<id>/automation.log` path) and provenance assertions. **Verified in this checkout:** `unittest tests.test_suite_verdict_gates` 32/32, full `unittest discover tests` **205/205**; and against the real archived logs, `batch4`/`batch5` classify `DID_NOT_COMPLETE` with "0 non-ensure crash report(s) ... 2 ensure-only" (reproducing the forensic pass automatically) while `batch6` classifies `COMPLETED_WITH_SKIPS` citing `automation.log:49733`. No Unreal build or suite run was performed. Docs updated: state tables in `CLAUDE.md` and `README.md`, the "should end with the marker" sentence replaced with what the checker enforces, and three `Docs/lessons.md` entries (truncated≠crashed, the three marker properties, verdicts must be citable). Remaining, deliberately NOT done here: migrating `CleanupTestAsset`/`DeleteAsset` callers off `ObjectTools::ForceDeleteObjects` (that is `B-suite-host-gc-crash-in-combined-group-run`), and the CI "Gate on test report" step still gates report counts separately.
