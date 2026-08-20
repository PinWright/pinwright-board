---
id: B-test-ids-swallowed-by-dot-prefix
title: "Five automation tests never execute because their ids are dot-extended prefixes of other ids — the engine turns the shorter id into a branch node and silently stops running it"
status: OPEN
severity: High
category: bug
tags: [tests, automation, test-id, prefix-collision, branch-node, silent-skip, coverage, suite-count, AutomationReport]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# A test id that another id extends with a `.` stops being a test

Five ids declared in the tree produce **no result of any kind** — not a pass, not a fail, not a skip.
Each is a strict prefix of a longer id that extends it with a `.`, and the engine's report tree
converts the shorter one from a leaf into a branch, after which it is never scheduled.

| Swallowed id | Declared at | Extended by |
|---|---|---|
| `PinWright.infra.handler_context.RequireAssetPath` | `Source/PinWright/Private/Tests/Infra/TestHandlerContext.cpp:349` | `.EnginePath` (`:551`), `.OnlySlashes` (`:571`) |
| `PinWright.niagara.decompile_nir.GraphParameterMapGet` | `Source/PinWright/Private/Tests/Niagara/TestNIRGraphDataflow.cpp:234` | `.MetadataType` (`:267`), `.AddPinSuppressed` (`:531`) |
| `PinWright.niagara.decompile_nir.GraphParameterMapSet` | `Source/PinWright/Private/Tests/Niagara/TestNIRGraphDataflow.cpp:309` | `.AddPinSuppressed` (`:567`) |
| `PinWright.niagara.reset_module_input` | `Source/PinWright/Private/Tests/Niagara/TestNiagaraResetModuleInput.cpp:36` | `.FindOverrideNodeNullGraph` (`:58`) |
| `PinWright.niagara.search_modules` | `Source/PinWright/Private/Tests/Niagara/TestNiagaraSearchModules.cpp:12` | `.StageAlias` (`:72`) |

None of the four files carries an `#if` guard, so this is not conditional compilation.

## Mechanism

`FAutomationReport::EnsureReportExists`
(`C:\UE_5.8\Engine\Source\Developer\AutomationController\Private\AutomationReport.cpp:580-676`)
splits a display name on the **first `.`** (`:587`) and looks up an existing child by
`GetFullTestPath()` (`:611`). Registering `A.B` first creates a **leaf** report (`:626`);
registering `A.B.C` afterwards matches that leaf at `:611-615` and, because the name remainder is
non-empty, **recurses into it** at `:672`, hanging `C` underneath. The reverse order is equally
fatal — `A.B.C` first creates a parent whose `TestName` is `TEXT("")` (`:631-632`), and `A.B` then
binds to that empty-named parent.

`GetNextReportToExecute` (`:679-694`) is the kill: `if (ChildReports.Num())` it iterates children
and **never returns itself**; only the `else` branch (`:695-713`) considers `AsShared()`.
`GetEnabledTestReports` (`:717-733`) has the identical shape. A node that acquired a child stops
being executable, and nothing on that path emits a warning, an ensure, or a log line.

## Empirical proof, from logs that already exist

`grep "Path={<exact-id>}"` over the host project's `Saved/Logs/`, no new run started:

| Log | `tests performed` | the five ids `Test Started` | their `.Child` ids |
|---|---|---|---|
| `pw_wave_suite.log` (2026-08-20 13:50, post-wave) | 4039 | **0 / 0 / 0 / 0 / 0** | all present |
| `pw_full_verify.log` | 3996 | **0** | all present |
| `pw_full_gate.log` | 3997 | **0** | all present |
| `pw_suite_orthotiles2.log` | 3844 | **0** | all present |

Four independent runs, five ids, zero starts.

## Only dot-extension collides

An exhaustive scan of the current tree finds 4,049 `IMPLEMENT_SIMPLE_AUTOMATION_TEST` sites, 4,049
unique ids, **zero** exact duplicates, and **77** strict-string-prefix pairs. 72 of those 77 are
letter-extensions (`...BindDispatcher` vs `...BindDispatcher_WithDelegate`, `...ConvertCppType_Int`
vs `...ConvertCppType_Int64`) and are harmless — both members start in `pw_wave_suite.log`. The
defect set is exactly the five ids above, because only a `.` reaches the split at `:587`.

## What the counts actually mean

The reported consequence needs one correction. 3844 / 3997 / 3996 / 4039 are counts of tests that
**executed**; the five never entered the queue, so they are not inside those numbers and the totals
are honest as run-counts. What is wrong is the **source-to-executed reconciliation**: the tree
declares five tests that yield no result, and the `CLAUDE.md` rule "derive expected = last measured
+ tests the diff adds" silently mis-derives whenever a newly added id dot-extends an existing one.
The failure is invisible in exactly the check built to catch missing tests.

## Lost coverage is real, not vacuous

`RequireAssetPath` carries 4 assertions including the `/Game/../../../etc/passwd` path-traversal
rejection at `TestHandlerContext.cpp:371-379` — a security assertion that has never run once.
`search_modules` has 8 assertions, `GraphParameterMapGet` 4, `GraphParameterMapSet` 4,
`reset_module_input` 2.

## Known in-tree, never swept

`Source/PinWright/Private/Tests/Infra/TestContractConsistency.cpp:442-448` documents this exact
mechanism with measured evidence — filtering on `PinWright.infra.contract` enumerated the six
`RequiredParamGate.Mechanism.*` leaves and ran 12 tests, with the bare-prefix walk missing and no
error anywhere in the log — and the author defensively named their own new test `.EveryVerb` to
dodge it. That comment landed in `c86f8890` (2026-08-20), inside the same wave. The five
pre-existing victims were not swept, and nothing structurally prevents the next one.

**Fix:** rename the five so each is a leaf (e.g. `RequireAssetPath` → `RequireAssetPathBasic`),
then add the structural guard: a contract-consistency test asserting that no registered test id is
a dot-prefix of another. The walk in `TestContractConsistency.cpp` already holds the registry, so
the guard is cheap and it is the half that stops this recurring.

## Related

- `B-test-skips-assertions-silently` (OPEN) — a test that *runs* and skips its assertions; the
  orthogonal half of "green count, no coverage". Neither ticket covers the other.
- `Docs/plans/defect-backlog.md` `D-8C` — vacuous tests that execute and cannot fail. Also
  orthogonal: those run, these do not.

## History
- `#1-five-ids-never-enter-the-queue` `OPEN` reporter — Five ids never enter the automation queue because each is a dot-extended prefix of another: `PinWright.infra.handler_context.RequireAssetPath` (`Tests/Infra/TestHandlerContext.cpp:349`), `PinWright.niagara.decompile_nir.GraphParameterMapGet` (`Tests/Niagara/TestNIRGraphDataflow.cpp:234`), `...GraphParameterMapSet` (`:309`), `PinWright.niagara.reset_module_input` (`Tests/Niagara/TestNiagaraResetModuleInput.cpp:36`), `PinWright.niagara.search_modules` (`Tests/Niagara/TestNiagaraSearchModules.cpp:12`). `FAutomationReport::EnsureReportExists` (`AutomationReport.cpp:580-676`, UE 5.8.1) splits display names on the first `.` (`:587`) and attaches the longer id as a *child* of the shorter id's leaf (`:611-615`, recursion at `:672`); `GetNextReportToExecute` (`:679-694`) returns a node only from its `else` branch when `ChildReports.Num() == 0`, so a node that acquired a child is skipped with no warning, ensure or log line. Proven empirically against four existing host-project logs — `pw_wave_suite.log` (4039 performed, post-wave), `pw_full_verify.log` (3996), `pw_full_gate.log` (3997), `pw_suite_orthotiles2.log` (3844) — each with zero `Test Started` lines for all five bare ids while every `.Child` id starts normally; no new run was started. Only dot-extension collides: of 77 strict-string-prefix pairs in the tree, the other 72 are letter-extensions and both members demonstrably run. Dead assertions include the `/Game/../../../etc/passwd` path-traversal rejection at `TestHandlerContext.cpp:371-379`, which has never executed. Correction to the original report: the suite totals are **not** five optimistic — the five never entered the queue, so they are absent from the counts, which are honest as run-counts; what breaks is the source-to-executed reconciliation and the `CLAUDE.md` "expected = last measured + tests the diff adds" rule, which mis-derives silently whenever an added id dot-extends an existing one. The mechanism is already documented in-tree at `TestContractConsistency.cpp:442-448` (landed `c86f8890`, same wave) and the author renamed their own test `.EveryVerb` to avoid it, but the five pre-existing victims were not swept. Fix: rename the five to leaves, and add a contract-consistency assertion that no registered id is a dot-prefix of another — the registry walk already has the data.
