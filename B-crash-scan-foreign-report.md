---
id: B-crash-scan-foreign-report
title: "check_suite_log attributes another editor's crash report to a clean run and classifies it CRASHED"
status: IN-REVIEW
severity: High
category: bug
tags: [testing, suite, check-suite-log, editor-test-status, crash-report, verdict, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T00:00:00Z
---

# check_suite_log attributes another editor's crash report to a clean run

Every editor of a project writes crash reports into the same `Saved/Crashes`. `scan_crash_reports`
(`Content/Python/mcp_proxy.py`, shared by `check_suite_log.check_log` and therefore by
`editor_test_status`) counted any non-ensure report whose `CrashContext.runtime-xml` mtime fell in
`[log mtime - run duration - 300 s, log mtime + 300 s]`, with no process check. The 300 s slack before
the run's start reached into earlier runs.

Repro (host `unreal-fpv`): run `8be87deef11f4bcaa0a3ebdbb43d6086`, log opened 15:40:05, 9/9 passed,
drain marker, exit 0, editor pid 57928. `editor_test_status` returned `CRASHED` on
`Saved/Crashes/UECC-Windows-624902454679BAEE01D81D97CDF4827B_0001`: an `Assert` written 15:38:28-15:38:37
by pid 64500 (run `ed6be6c8dd3543b49ee4de940a39a39b`, whose `-Abslog` its `<CommandLine>` names).

**Fix:** a report counts only inside the run's own window, from the log's `Log file open` header
(local time, same clock as mtimes; fallback: mtime minus the run's UTC-stamp duration) to the log's
last write plus 300 s; only when the `-Abslog=` its `<CommandLine>` records (if any) is this log;
and, when both are known, only if its `<ProcessId>` equals the run's editor pid.
The pid comes from `check_log(pid=...)` / `check_suite_log --pid`, else from the `started pid N:` line
of `<log>.supervisor.log` (written at start, so it survives a supervisor that never wrote its verdict).
A bare log still works: window plus Abslog.

## History
- `#1-foreign-report-false-crash` `OPEN` reporter — Run 8be87dee (clean 9/9, pid 57928) classified CRASHED on pid 64500's assert report written 90 s before its log opened; the scan's pre-start slack and missing pid check are the cause, in `scan_crash_reports`, not in how `editor_test_status` or the supervisor pass the window (they pass only the log path).
- `#2-window-from-log-open-and-pid-match` `IN-REVIEW` developer — `mcp_proxy.py`: `parse_automation_log` adds `openedAt` from the `Log file open` header; `scan_crash_reports(started_at, pid)` starts the window there with no pre-start slack and drops reports whose `<ProcessId>` differs from a known run pid (`_read_crash_context` returns the pid). `pinwright_supervisor.read_started_pid` reads the pid from `<log>.supervisor.log`; `check_suite_log.check_log(pid=None)` uses it by default, `--pid` overrides, and the `crashReports:` line prints the pid. Tests in `tests/test_suite_verdict_gates.py`: report before the log opened (pid-less) -> COMPLETED_CLEAN; report from pid 64500 inside the window -> COMPLETED_CLEAN, CRASHED again once the pid is unknown, clean with `pid=`; the run's own pid report -> CRASHED. Real logs: 8be87dee now COMPLETED_CLEAN `(pid 57928)`, ed6be6c8 still CRASHED on its own report `(pid 64500)`. Python suite 342 OK (1 skipped).
- `#3-abslog-match` `IN-REVIEW` developer — Added the `-Abslog` rule: `_read_crash_context` also returns the `-Abslog=` value from the report's XML-escaped `<CommandLine>` (quoted or bare); a report whose Abslog differs from the classified log (`_same_path`, normcase, so case-insensitive on Windows) is another run's regardless of pid, and one whose Abslog matches counts even with no pid known (the pid rule still applies when both pids are known). Head read raised to 16 KB so a long command line stays inside it. Tests: other Abslog with the run's own pid, with and without the supervisor log -> COMPLETED_CLEAN; same Abslog (upper-cased on Windows), no pid anywhere -> CRASHED. Real logs unchanged: 8be87dee COMPLETED_CLEAN, ed6be6c8 CRASHED on its own report. Python suite 344 OK (1 skipped).
