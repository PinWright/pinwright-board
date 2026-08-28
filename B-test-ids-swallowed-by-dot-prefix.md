---
id: B-test-ids-swallowed-by-dot-prefix
title: "Adding a `.Suffix` test id under an id that is already a complete test kills that existing test — the engine turns the shorter id into a branch node and silently stops running it"
status: IN-REVIEW
severity: High
category: bug
tags: [tests, automation, test-id, prefix-collision, branch-node, silent-skip, coverage, suite-count, AutomationReport]
encounters: 2
lastSeen: 2026-08-28T00:00:00Z
---

# A test id that another id extends with a `.` stops being a test

## The rule, and the direction everyone gets wrong

Two automation ids collide when one is the other plus `.` plus more. When a pair collides, **the
SHORTER id is the one that dies** — regardless of which of the two was written first, and with no
pass, no fail, no skip, no warning, and no entry in the `<N> tests performed` count.

- **Forward — the direction people check.** You add `A.B`; `A.B.C` already exists. Your own new id
  is the one that breaks, so you notice.
- **Reverse — the direction that keeps reproducing.** You add `A.B.C`; `A.B` already exists as a
  complete leaf. **Your new test runs fine. The existing `A.B` stops running, permanently.** Your
  own id is clean by every check you thought to make, and nothing in the run points at what you
  broke: the victim leaves the queue rather than failing in it.

So *"I checked that my id is not a prefix of anything"* is half a check. The other half is *"my id
does not dot-extend anything that already exists"*. Every agent who claimed to check for collisions
on 2026-08-28 checked only the forward direction, which is exactly how
`PinWright.niagara.graph.create_node` was killed by a new
`…create_node.RefusalLeavesPackageClean` (see `#4-sixth-victim-caught-at-runtime`).

**Fix, whichever direction it came from: rename the SHORTER id to a leaf** naming what it asserts,
and update every reference to the old id.

Only a `.` collides. `create_node` vs `create_node_guard` is a plain string prefix that never
reaches the engine's split on `.`; both members run and neither needs renaming.

## The five original victims — all renamed since; kept for the mechanism they proved

Five ids declared in the tree produced **no result of any kind** — not a pass, not a fail, not a skip.
Each was a strict prefix of a longer id that extends it with a `.`, and the engine's report tree
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

**Fix — state as of `#5-static-scan-added`.** The renames are done (the five above plus the sixth,
`niagara.graph.create_node` → `…create_node.PayloadApplicationAndErrorCodes`), and the structural
half is now two checks that between them cover both blind spots. Neither subsumes the other:

| | Runtime | Static |
|---|---|---|
| What | `PinWright.infra.automation_registry.NoPrefixCollisions` (`Source/PinWright/Private/Tests/Infra/TestAutomationTestIdPrefixCollisions.cpp`) | `Plugins/PinWright/Content/Python/check_test_ids.py` |
| Reads | `FAutomationTestFramework::GetValidTestNames` | source text, `IMPLEMENT_*AUTOMATION_TEST*` id literals |
| Needs | a live editor on this host | nothing — stdlib Python, no engine, no build |
| Blind to | ids inside a `#if` this host compiles out; every id of an integration sub-module whose engine plugin is disabled here | nothing conditional — but it can over-report a pair no single build would register together, which is the safe direction |
| Runs as | part of the suite | its own `test-id-scan` CI job, ahead of the compile legs |

The static scan does not evaluate the preprocessor and deliberately does not pretend to: it
collects every id literal, guarded or not. It does strip comments, because `//` and `/* */` are
lexical rather than preprocessor state, and a commented-out macro is not a registration. It fails
closed on a vacuous scan (zero ids found) and on any macro site whose id literal it cannot read,
since an unparsed site is an unchecked id. Its own fixture pair —
`Content/Python/tests/test_test_id_prefix_scan.py` — proves it flags a real dot-extension and does
**not** flag a `create_node_guard` letter-extension, so a scan that has quietly stopped being able
to fail cannot pass for a clean tree.

## Related

- `B-test-skips-assertions-silently` (OPEN) — a test that *runs* and skips its assertions; the
  orthogonal half of "green count, no coverage". Neither ticket covers the other.
- `Docs/plans/defect-backlog.md` `D-8C` — vacuous tests that execute and cannot fail. Also
  orthogonal: those run, these do not.

## History
- `#1-five-ids-never-enter-the-queue` `OPEN` reporter — Five ids never enter the automation queue because each is a dot-extended prefix of another: `PinWright.infra.handler_context.RequireAssetPath` (`Tests/Infra/TestHandlerContext.cpp:349`), `PinWright.niagara.decompile_nir.GraphParameterMapGet` (`Tests/Niagara/TestNIRGraphDataflow.cpp:234`), `...GraphParameterMapSet` (`:309`), `PinWright.niagara.reset_module_input` (`Tests/Niagara/TestNiagaraResetModuleInput.cpp:36`), `PinWright.niagara.search_modules` (`Tests/Niagara/TestNiagaraSearchModules.cpp:12`). `FAutomationReport::EnsureReportExists` (`AutomationReport.cpp:580-676`, UE 5.8.1) splits display names on the first `.` (`:587`) and attaches the longer id as a *child* of the shorter id's leaf (`:611-615`, recursion at `:672`); `GetNextReportToExecute` (`:679-694`) returns a node only from its `else` branch when `ChildReports.Num() == 0`, so a node that acquired a child is skipped with no warning, ensure or log line. Proven empirically against four existing host-project logs — `pw_wave_suite.log` (4039 performed, post-wave), `pw_full_verify.log` (3996), `pw_full_gate.log` (3997), `pw_suite_orthotiles2.log` (3844) — each with zero `Test Started` lines for all five bare ids while every `.Child` id starts normally; no new run was started. Only dot-extension collides: of 77 strict-string-prefix pairs in the tree, the other 72 are letter-extensions and both members demonstrably run. Dead assertions include the `/Game/../../../etc/passwd` path-traversal rejection at `TestHandlerContext.cpp:371-379`, which has never executed. Correction to the original report: the suite totals are **not** five optimistic — the five never entered the queue, so they are absent from the counts, which are honest as run-counts; what breaks is the source-to-executed reconciliation and the `CLAUDE.md` "expected = last measured + tests the diff adds" rule, which mis-derives silently whenever an added id dot-extends an existing one. The mechanism is already documented in-tree at `TestContractConsistency.cpp:442-448` (landed `c86f8890`, same wave) and the author renamed their own test `.EveryVerb` to avoid it, but the five pre-existing victims were not swept. Fix: rename the five to leaves, and add a contract-consistency assertion that no registered id is a dot-prefix of another — the registry walk already has the data.
- `#2-additional-target-split` `OPEN` reporter — Additional evidence: **Adversarial review A — VERIFY FIRST; the defect remains in the review-target checkout, while a separate UE 5.8 host clone contains a landed fix and runtime proof.** Actuality: PARTIAL. Framing: the UE 5.8 report-tree mechanism remains confirmed, and the five old IDs are still declared in `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Infra\TestHandlerContext.cpp:349`, `...\\Niagara\\TestNIRGraphDataflow.cpp:234,309`, `...\\TestNiagaraResetModuleInput.cpp:36`, and `...\\TestNiagaraSearchModules.cpp:12`, so High remains valid for builds from that checkout. However, active host-plugin history has fix `0064cc61` on HEAD `8ac45d5e`: its five IDs are leaf-renamed and it adds a case-insensitive registry guard. Proposed fix: SYSTEMIC, the landed change fixes every named victim and adds a self-checking `GetValidTestNames`/`GetFullTestPath` prefix guard; the guard's documented current-host/filter/submodule limits and the absence of a source-level scan in CI leave a delivery/configuration gap. Evidence: `X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWright\Private\Tests\Infra\TestAutomationTestIdPrefixCollisions.cpp:62-78,81-175`, `X:\src\unreal\EAContentExamples58\Plugins\PinWright\.github\workflows\ci.yml:133-156,179-215`, `X:\src\unreal\EAContentExamples58\Saved\Logs\pw_final_suite.log:28902,29362-29367,35821-35841,36168-36182`. Runtime: verified in the existing final suite: guard reports 0 PinWright collisions and all five renamed tests start and complete successfully; queue drained at 4242 tests. Recommendation: VERIFY FIRST; identify the shipping checkout, propagate `0064cc61` (or equivalent) to it, add a host-independent static prefix scan to CI, and close the candidate only after target runtime/CI evidence is clean.
- `#3-additional-target-still-live` `OPEN` reporter — Additional evidence: **Adversarial review B — A's split finding survives, but its `PARTIAL` label undercalls the review target: a fix in another clone does not fix this checkout.** Actuality: **CONFIRMED CURRENT**. Framing: the five bare ids remain in both `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Infra\TestHandlerContext.cpp:349`, `...\\Niagara\\TestNIRGraphDataflow.cpp:234,309`, `...\\TestNiagaraResetModuleInput.cpp:36`, and `...\\TestNiagaraSearchModules.cpp:12` and the canonical `X:\src\unreal\unreal-fpv` clone; the target plugin lacks `0064cc61` and its `TestContractConsistency.cpp` has no prefix guard. UE 5.8 still adopts a matching leaf as a parent (`C:\UE_5.8\Engine\Source\Developer\AutomationController\Private\AutomationReport.cpp:580-713`). The separate `EAContentExamples58` fix is real, but only its renamed tests/guard are covered; its guard explicitly excludes feature-filtered and disabled sub-module tests. Target CI runs `Automation RunTests PinWright; Quit` and only checks report `failed/notRun/inProcess/total` (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\.github\workflows\ci.yml:80-116`), with no source-wide reconciliation or static prefix scan, so swallowed ids can still be absent without failing. Proposed fix: **INCOMPLETE**, the landed rename/host registry walk is systemic for one host but not delivery-wide; propagate it to target/canonical and add a host-independent scan. Evidence: `X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWright\Private\Tests\Infra\TestAutomationTestIdPrefixCollisions.cpp:62-175`, `X:\src\unreal\EAContentExamples58\Saved\Logs\pw_final_suite.log:28862-28903,29356-29377,35815-35843,36161-36190,45180-45182`. Runtime: target NOT VERIFIED; the existing EA58 log verifies only the fixed clone (0 PinWright collisions, all five renamed tests complete, 4242 queued). Recommendation: KEEP; propagate the fix, add the static/CI guard, then run target and canonical suites before any close transition.
- `#4-sixth-victim-caught-at-runtime` `OPEN` reporter — 2026-08-28, `unreal-fpv-new` checkout. The runtime guard this ticket asked for now exists here (`Source/PinWright/Private/Tests/Infra/TestAutomationTestIdPrefixCollisions.cpp`) and caught a sixth victim on its first recurrence: `PinWright.infra.automation_registry.NoPrefixCollisions` failed with exactly one Error — `PinWright.niagara.graph.create_node` (declared `Tests/Niagara/TestNiagaraGraphCreateNode.cpp:27`) adopted as a branch node by `PinWright.niagara.graph.create_node.RefusalLeavesPackageClean` (`:211`), added in the same session as the regression test for `B-niagara-refused-edit-dirties-package` `#3`. **The new-test author checked only that their own id was not a prefix of an existing one, not the reverse direction — that a NEW id turns an EXISTING leaf into a branch.** That direction is the one the rename-the-five framing above does not make obvious, and it is how this defect reproduces indefinitely: any suffixed test added under an id that is itself a complete leaf silently kills that leaf. Fixed by renaming the pre-existing bare id to `PinWright.niagara.graph.create_node.PayloadApplicationAndErrorCodes` (it asserts `ApplyCreateNodePayload`'s per-class payload routing plus the `INVALID_OP` / `UNSUPPORTED_NODE_CLASS` / `CLASS_NOT_FOUND` / `INVALID_ARGUMENT` codes); the only other in-tree reference was a prose mention in `Tests/Niagara/TestNiagaraGraphCreateNodeCreatorGuard.cpp:27`. The `_guard` id `PinWright.niagara.graph.create_node_guard.RejectionLeavesGraphUnchanged` is a letter-extension, not a dot-extension, and was correctly never at risk. A static sweep of all 4629 `IMPLEMENT_*AUTOMATION_TEST` ids under `Source` (gitignored `dist/` excluded) now finds zero dot-prefix pairs and zero duplicate ids. Not re-run here — the recompile and re-run are owned by the orchestrating session. Status stays `OPEN`: the host-independent static/CI scan asked for in `#2`/`#3` still does not exist in this checkout, so the guard remains runtime-only and a host with an integration sub-module disabled still cannot see collisions among that sub-module's ids.
- `#5-static-scan-added` `IN-REVIEW` developer — Both remaining gaps closed in `unreal-fpv-new`. (1) **Framing.** The ticket led with "rename these five ids" and never stated the reverse direction, which `#4` identified as the reason every agent's collision check that day was half a check. The title and the body now lead with it: a new `.Suffix` id added under an id that is already a complete leaf kills that leaf, the shorter id always being the one that dies. (2) **Host-independent static scan**, new: `Plugins/PinWright/Content/Python/check_test_ids.py` — stdlib-only, no editor, no engine, no build, mirroring `check_suite_log.py`'s house shape. It walks all modules (excluding `dist/`, `Binaries/`, `Intermediate/`, `Saved/`, `.git/`), strips comments with a literal-aware lexer, parses the id out of every `IMPLEMENT_*AUTOMATION_TEST*` spelling (the first string literal in the argument list, which is the pretty name in all of SIMPLE / COMPLEX / CUSTOM_* / NETWORKED / BDD / _PRIVATE), and fails on any dot-prefix pair or duplicate id, folding case the way `FString::operator==` does. Conditional compilation is **not** evaluated and the file says so: every id literal is collected whether or not any host compiles it, so unlike the runtime guard it cannot miss an id behind a false `#if` or in a disabled sub-module, at the cost of being able to flag a pair no single build would register together — the safe direction, since a dot-prefix name is a defect in the tree whichever host compiles it. Comments *are* stripped, because `//` and `/* */` are lexical rather than preprocessor state. It fails closed on a vacuous scan (zero ids found greps identically to a clean one) and on any macro site whose id literal it cannot read, since an unparsed site is an unchecked id. **Measured on this checkout: 4637 ids, 4637 unique, 843 files, zero dot-prefix collisions, zero duplicates, exit 0** — a figure that moved 4632 → 4637 within the same hour as other sessions landed tests, so re-measure rather than quoting it. Cross-checked against `grep -rc IMPLEMENT_*AUTOMATION_TEST Source` = 4637 exactly, which is what proves the `dist/` exclusion (it holds a whole second copy of the tree) and the parser agree. **Self-test**, new: `Content/Python/tests/test_test_id_prefix_scan.py`, 14 tests, all passing under the bundled interpreter — the required fixture pair (a real `…create_node` + `…create_node.RefusalLeavesPackageClean` collision flagged; a `…create_node_guard.…` letter-extension not flagged) plus a three-id case proving the sorted walk steps *past* a letter-extension instead of stopping at it, engine-style case folding, duplicates, `#if`-guarded ids still seen, commented-out macros ignored, ids that are only string data (the skip-marker fixtures' shape) not counted as registrations, a `//` inside a string literal not eating the file, excluded build dirs, unparsed-macro-is-an-error, empty-tree-is-not-green, all four other macro spellings, and a non-vacuity + cleanliness check against the real tree. **CI**, new `test-id-scan` job in `.github/workflows/ci.yml`: one job for the whole dispatch rather than per matrix leg, no engine required, self-test run *before* the scan so the verdict is only believed from a scanner proven able to fail. Docs: `Docs/test-organization.md`'s "Every test id must be a leaf" section now leads with the reverse direction and tabulates both checks; the plugin `CLAUDE.md` suite-reconciliation note and the runtime guard's own LIMITATIONS comment both point at the scan (the comment previously described it as hypothetical). **The runtime guard was not weakened, narrowed or replaced** — only its trailing comment changed. Not compiled and no suite run: the C++ edit is comment-only, and the Python was verified by running it. Tester should re-run the scan (`python -m check_test_ids` from `Content/Python`) and `python -m unittest tests.test_test_id_prefix_scan`, and confirm the `test-id-scan` job passes on a dispatch.
