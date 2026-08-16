---
id: B-suite-log-completeness-unverifiable
title: "A killed automation suite produces a log that greps clean — zero failures, no crash, no completion marker — and is indistinguishable from a green run; two such logs were quoted today as evidence of green suites, and one widely-quoted suite figure has no log on disk at all"
status: OPEN
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
