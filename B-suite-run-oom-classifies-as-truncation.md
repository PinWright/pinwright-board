---
id: B-suite-run-oom-classifies-as-truncation
title: "A suite run that exhausts memory is unbounded and unrecognisable — no cap on the editor's working set, and check_suite_log classifies the OOM as DID_NOT_COMPLETE"
status: IN-REVIEW
severity: High
category: bug
tags: [automation-suite, memory, oom, garbage-collection, check-suite-log, verdict, ci, job-object]
encounters: 1
costly: 0
lastSeen: 2026-09-09T17:45:00Z
---

# Two gaps, one failure mode

**No bound.** Every caller of the PinWright suite — `.polyskill/skills/mcp-test-loop/SKILL.md`,
`.polyskill/skills/mcp-version-matrix/mcp-version-matrix.workflow.js`,
`.github/workflows/ci.yml` — launched the identical argv and none of them constrained the
editor's memory. On a 63 GB host the suite peaks around 15 GiB, so nothing was visibly wrong;
the same absence of a bound is what let `asset.dump_folder` grow to
`UsedVirtual 128.06 GiB` and take the process with it
(`B-dump-folder-sweep-never-gcs-ooms-editor`). The suite's own reclaim was scheduled on a pure
test-count modulo (`pinwright.TestGcEvery`, default 25) with no memory input at all, so a run
whose fixtures are heavier than average walks past its headroom between resets and the count
cannot notice.

**No verdict.** `check_suite_log.py` had six states and none of them was memory. An OOM'd
editor leaves exactly a truncation's log — it stops, the queue never drains, and the allocator
often gives up before any fatal banner reaches the file — so the run classified
`DID_NOT_COMPLETE`, i.e. "killed or wedged". That names the symptom and points the next reader
at a harness timeout or another agent's build closing the editor, both of which have really
happened on this tree and are documented as such in `Plugins/PinWright/CLAUDE.md`. The two
engine strings that would have settled it (`Ran out of memory allocating`,
`Freeing ... from backup pool to handle out of memory`) were grepped by nothing.

## Fix

Two layers, one committed each.

**External cap — `scripts/Run-SuiteCapped.ps1`** (commit `a8913df5`). Builds the identical
suite argv and creates the editor suspended inside a Windows Job Object with
`JOB_OBJECT_LIMIT_PROCESS_MEMORY | JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` at `-MemoryFraction`
(default 0.60) of physical RAM, assigns it, then resumes — suspended-first because UE reads
the job limit once, at startup, and a running process assigned late has already allocated.
Three properties that a polling watchdog does not have:

- UE folds a job memory limit into `MemoryConstants.TotalVirtual`
  (`WindowsPlatformMemory.cpp:449-486`), so `FAssetCompilingManager` throttles against the cap
  rather than against the whole box. **Verified on the host:** the capped run logs
  `LogMemory: Process is running as part of a Windows Job with separate resource limits` and
  `Memory total: Physical=63.2GB (64GB approx) Virtual=37.9GB` — 37.9 GB is the cap.
  Note the engine's own `Detected a per-process memory limit of %.1fGB for this job.` line is
  emitted during memory init, before `GLog` has a file, so it never reaches `-Abslog`; the
  launcher keys on the two lines above instead.
- The limit is **per-process, not job-wide**. `ShaderCompileWorker` children join the job and
  would be charged to the editor's budget under `JOB_OBJECT_LIMIT_JOB_MEMORY`, failing the
  editor for someone else's allocation.
- A hard per-process limit makes allocations **fail** rather than killing the process, so the
  run ends at a known bound with the engine's OOM strings in the log instead of paging the host
  to a standstill.

It prints, and writes to `<LogPath>.result.txt`, one machine-readable line:
`PINWRIGHT_SUITE_RESULT verdict=... cap_gb=... peak_gb=... exit=... oom_alloc=... oom_backup_pool=... watermark_markers=... capSeenByEditor=... wall_min=... log=...`,
with `verdict=MEMORY_CAP_HIT` and exit 2 when the cap tripped. Numbers are formatted invariant
— the first draft printed `cap_gb=37,91` on this ru-locale host, a decimal comma that reads as
a thousands separator to every parser downstream. It does **not** classify the suite;
`check_suite_log.py` stays the verdict authority.

**Internal watermark — `PinWrightSuiteMaintenance`** (commit `2318c0e4`). `AdvanceAndShouldReset`
now delegates to `AssetDumpHandler::ShouldRunDumpReleaseStep` — the same predicate the folder
sweep's release step uses, so "this long-running loop has accumulated enough to be worth
reclaiming" has one rule — with `TestsSinceReset`, the resolved interval, `UsedPhysical`,
`TotalPhysical` and a new watermark. Resolution mirrors the interval's chain: scoped override →
`pinwright.TestMemoryWatermark` → `-PinWrightTestMemoryWatermark=F` → the new
`UPinWrightSettings::TestSuiteResetMemoryWatermark` (0.55, deliberately under the 0.60 cap so
the in-process reclaim runs before an allocation can fail). Interval and hard fraction get the
same treatment (`TestSuiteResetIntervalTests` 25, `TestSuiteMemoryHardFraction` 0.75).

Every reset now logs at **Display**, not Verbose, with its trigger and before/after GiB — a
reset that never happened and a reset that reclaimed nothing were previously the same silence.
When the working set is still at or above the hard fraction *after* the collect, the run emits
`PINWRIGHT_MEMORY_WATERMARK_EXCEEDED` at Error with the last completed test id.

**Checker.** `MEMORY_EXHAUSTED` (on the two engine OOM strings **and** a queue that did not
drain) and `COMPLETED_WITH_MEMORY_PRESSURE` (just above `COMPLETED_CLEAN`, on the watermark
token or an allocation failure the run survived). Both exit non-zero. Every verdict now prints
`memory: oomLines=N watermarkMarkers=N` on every run, zeros included, for the same reason the
crash line does: an unchecked axis reported as silence is what let an OOM read as a truncation.

**`MEMORY_EXHAUSTED` outranks `CRASHED`, and that was a correction.** The first cut ranked it
between `CRASHED` and `DID_NOT_COMPLETE` on the assumption that an OOM'd editor stops without a
banner. It does not: the capped verification run below logged the OOM strings, then died with
`Fatal error:` and wrote a crash report whose type is literally `OutOfMemory`, so the new state
never fired and the run classified `CRASHED` — correct, uninformative, and it points at
`Saved/Crashes` rather than at memory. The reorder folds the banner and the crash report into
the memory verdict's own reason, and a control test pins that a fatal banner with **no**
allocation failure still reads `CRASHED`.

## Two traps worth keeping

**A test must not emit the real marker.** The first draft's escalation test drove
`RunResetNow` with an injected working set and asserted the counter. It worked, and it put
`PINWRIGHT_MEMORY_WATERMARK_EXCEEDED` into the suite's own log, which would have classified
**every** run `COMPLETED_WITH_MEMORY_PRESSURE`. The escalation is now covered by a pure
predicate plus a structural contract that reads `RunResetNow`'s own source and asserts the
guard, the counter, the Error verbosity and that `mcp_proxy.py` greps the same literal.

**`UE_LOG(..., Error, ...)` from `OnTestEnd` is safe, but only just.**
`FAutomationTestFramework::InternalStopTest` broadcasts `OnTestEndEvent` *after* freezing the
finished test's success state (`AutomationTest.cpp:1384`) but *before* detaching the automation
output device (`:1407`), so the line lands as an Error **event** on that test's report and
cannot turn its `Result={Success}` into a failure. Inside a *running* test the same log fails
it, which is why any test that deliberately triggers it needs `AddExpectedError`.

## Measured

Host `X:\src\unreal\unreal-fpv-dev`, UE 5.8, 63.18 GiB RAM, cap 37.91 GiB, full `PinWright`
filter through the new launcher, plugin at `2318c0e4`:

| run | tests | pass | fail | skips | resets (count/manual/watermark) | peak | wall |
|---|---|---|---|---|---|---|---|
| `-PinWrightTestGcEvery=5` (`Saved/Logs/pw_oom_gc5.log`) | 5258 | 5257 | 1 | 95 | 1054 (1052/1/1) | 16.73 GiB | 11m38s |
| `-PinWrightTestGcEvery=25` (`Saved/Logs/pw_oom_gc25.log`) | 5258 | 5257 | 1 | 95 | 212 (210/1/1) | 15.02 GiB | 10m51s |

Both: `started == succeeded + failed`, drain marker present, `capSeenByEditor=True`, 0 OOM
lines, 0 real watermark markers. The one `manual` and one `watermark` reset in each run are the
two contract tests calling `RunResetNow` directly, not memory events. The single failure,
`PinWright.audio.authoring.set_sound_wave_properties.PublishesOrdinaryProperties`, is
**identical at both intervals**, so nothing in the suite is GC-timing-dependent at 5× the reset
rate — and it is pre-existing at `b694c7a4` on a working tree that touches no audio code. It
is not part of this ticket.

**The cap tripped on purpose, and the whole chain was observed end to end.** Same host, filter
`PinWright.infra`, `-MemoryFraction 0.05` (cap 3.16 GiB), `Saved/Logs/pw_oom_capped.log`. The
editor read the cap (`Memory total: Physical=63.2GB (64GB approx) Virtual=3.2GB`), ran for 17 s,
logged 2 `Ran out of memory allocating` and 1 backup-pool line, died with `Fatal error:` and an
`OutOfMemory` crash report, and never started a test. Launcher:
`verdict=MEMORY_CAP_HIT cap_gb=3.16 peak_gb=3.16 exit=3 oom_alloc=2 oom_backup_pool=1`, exit 2 —
the peak pinned exactly at the cap is the second, independent read that does not need the log.
Checker: `MEMORY_EXHAUSTED`, exit 1, reason `the editor ran out of memory and the queue had NOT
drained: 3 allocation-failure line(s), log banner 'Fatal error:', crash report
UECC-Windows-F21EF6FF46E5ECB0C793388E8743BCE7_0000 (OutOfMemory)`.

**Note on the min-gap floor.** `ShouldRunDumpReleaseStep` will not fire its watermark branch
before `DumpReleaseMinAssetsBetweenSteps` (25) items, so at the default interval of 25 the
watermark can never fire *earlier* than the count. It earns its keep when the interval is
raised or disabled. Reusing the predicate was still the right call over a near-duplicate.

## History
- `#1-suite-oom-unbounded-and-misclassified` `IN-REVIEW` developer — Filed and fixed in the same pass, so this never sat `OPEN`. Two gaps: no memory bound on any suite launch (all three callers shipped the same unconstrained argv, and the in-process reset was a pure count modulo with no memory input), and no verdict for an OOM (`check_suite_log.py` had no state keyed on `Ran out of memory allocating` / `from backup pool to handle out of memory`, so an OOM'd editor — a log that stops with no drain marker and usually no fatal banner — classified `DID_NOT_COMPLETE`, naming the symptom). Fixed in two commits on `origin/master`: `a8913df5` adds `scripts/Run-SuiteCapped.ps1` (Job Object, `JOB_OBJECT_LIMIT_PROCESS_MEMORY | KILL_ON_JOB_CLOSE`, suspended-create → assign → resume, default 0.60 of physical RAM, `PINWRIGHT_SUITE_RESULT` line, exit 2 on cap hit), the two new checker states, and repoints all three callers plus `Docs/test-organization.md` § Suite Memory Containment; `2318c0e4` adds `TestSuiteResetIntervalTests` / `TestSuiteResetMemoryWatermark` / `TestSuiteMemoryHardFraction`, routes `AdvanceAndShouldReset` through `AssetDumpHandler::ShouldRunDumpReleaseStep`, logs every reset at Display with its trigger and before/after GiB, and escalates to `PINWRIGHT_MEMORY_WATERMARK_EXCEEDED` when a collect leaves the working set above the hard fraction. Verified on the host: the cap reaches the editor (`Memory total: ... Virtual=37.9GB`), two full-suite runs at reset intervals 5 and 25 are 5258/5257/1 with identical fail sets (so no test is GC-timing-dependent), peaks 16.73 and 15.02 GiB against a 37.91 GiB cap, wall 11m38s and 10m51s. Python side 234/234 `unittest discover`, `check_test_ids` and `check_test_skips` clean, `check_unattended_flags` clean over 11 command files. **For the tester:** `COMPLETED_WITH_MEMORY_PRESSURE` has still never been produced by a real event — it is exercised only by the Python fixtures plus the structural contract, because no run on this host has left the working set above 0.75 of RAM after a collect. `MEMORY_EXHAUSTED` no longer has that gap (see `#2`).
- `#2-cap-tripped-and-the-ranking-was-wrong` `IN-REVIEW` developer — Deliberately tripped the cap rather than leaving it to a tester, and it found a real defect in `#1`. Same host, filter `PinWright.infra`, `-MemoryFraction 0.05` (cap 3.16 GiB): the editor read the cap (`Memory total: ... Virtual=3.2GB`), ran 17 s, logged 2 `Ran out of memory allocating` + 1 backup-pool line, and died with `Fatal error:` plus a crash report of type `OutOfMemory`. The launcher reported `verdict=MEMORY_CAP_HIT cap_gb=3.16 peak_gb=3.16 exit=3 oom_alloc=2 oom_backup_pool=1` and exit 2 — correct — but `check_suite_log.py` returned **`CRASHED`**, because `#1` ranked `MEMORY_EXHAUSTED` *below* the crash rule on the assumption that an OOM'd editor stops without a banner. It does not, so the new state would essentially never have fired in production and the verdict would have kept naming the wrong axis, just a different wrong one than before. Fixed in `cce27c0d`: the memory rule now precedes the crash rule, is gated on a queue that did NOT drain, and folds the banner and crash-report evidence into its own reason; an allocation failure on a run that *drained* (the backup pool absorbing one) falls through to `COMPLETED_WITH_MEMORY_PRESSURE` instead, since such a run measured everything. Two tests pin the pair: the reorder itself and a control asserting a fatal banner with no allocation failure still reads `CRASHED`. Re-classifying the same capped log now yields `MEMORY_EXHAUSTED`, exit 1, reason `the editor ran out of memory and the queue had NOT drained: 3 allocation-failure line(s), log banner 'Fatal error:', crash report ... (OutOfMemory)`. Python suite 236/236. `Docs/test-organization.md`, `CLAUDE.md` and the `mcp-test-loop` skill carry the corrected ordering and the measurement that earned it.
