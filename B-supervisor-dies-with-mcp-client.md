---
id: B-supervisor-dies-with-mcp-client
title: "The capped supervisor and its editor die with the MCP client's process tree on Windows, so a 'detached' suite run is killed mid-queue when the Claude Code process exits"
status: IN-REVIEW
severity: High
category: bug
tags: [proxy, supervisor, editor-run-tests, windows, detach, wmi, gap-analysis-2026-09-28]
encounters: 1
costly: 1
lastSeen: 2026-09-29T00:00:00Z
---

# The capped supervisor dies with the MCP client's process tree

`spawn_supervised` started the supervisor as a direct child of the MCP proxy (`DETACHED_PROCESS |
CREATE_NEW_PROCESS_GROUP | CREATE_NO_WINDOW`, first with `CREATE_BREAKAWAY_FROM_JOB`, silently
retried without it when the caller's job refused breakaway). None of those flags changes the
parent pid, so the chain stays `claude.exe -> mcp_proxy.py -> supervisor -> editor`, and a tree
kill of the client (`taskkill /PID <pid> /T /F`, which Claude Code's Windows teardown uses) reaches
the supervisor. The supervisor's own Job Object is `KILL_ON_JOB_CLOSE`, so its death takes the
editor with it. The docs promised "the run outlives the proxy and the MCP client".

Evidence (full suite run `8579e05411884222ab0c0b1b719a670e`, editor pid 67296, host `unreal-fpv`):
- `automation.log.supervisor.log` holds only the `started pid 67296` line: no exit line, no
  `PINWRIGHT_SUITE_RESULT`, so the supervisor was killed, not the editor alone. No crash report, no
  OOM-watchdog kill.
- The log's last write is 16:34:26.80 local, mid-test (`PinWright.niagara.create_system.PathIsGuardedBeforeCreatePackage` started).
- The launching subagent (`agent-a03c466a511fbae82`, session `be713d41`) had a Bash wait loop on
  pid 67296 SIGKILLed (`Exit code 137`) at 13:34:25.859Z; its transcript ends at 13:34:25.978Z.
  The resumed session's task notification says it "didn't finish before the previous session
  ended ... may have been running when the previous Claude Code process exited". So the Claude
  Code process exited at 13:34:25Z, and the editor died 0.9 s later.
- Job hypothesis checked and not the cause here: every process on this machine, `explorer.exe`
  included, is in a job whose innermost flags are `0x1800` (`BREAKAWAY_OK | SILENT_BREAKAWAY_OK`, no
  `KILL_ON_JOB_CLOSE`), and a `CREATE_BREAKAWAY_FROM_JOB` spawn from inside a Claude Code session
  succeeds. So breakaway was not refused here; the parent-pid tree is what carried the kill. The
  silent retry without breakaway was still a real gap on hosts whose job forbids breakaway.

**Fix:** on Windows the supervisor is started through WMI `Win32_Process.Create` (powershell
`Invoke-CimMethod`, `DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP`, hidden window): `WmiPrvSE` is its
parent, so it is outside the client's process tree and job. A WMI failure falls back to the direct
child and the result says so (`detached: false`, `detachNote`, `NOT DETACHED` in the text). Linux
is unchanged (direct child, new session).

## History
- `#1-suite-killed-with-client` `OPEN` reporter — Full suite 8579e054 died mid-queue together with its supervisor at the moment the Claude Code process that owned the MCP proxy exited; the supervisor was a parent-pid descendant of the proxy, and the breakaway fallback degraded silently.
- `#2-wmi-launch` `IN-REVIEW` developer — `pinwright_supervisor.py`: `_start_supervisor` launches through `_wmi_create` (WMI `Win32_Process.Create` via powershell, command line passed in the environment) and falls back to `_supervisor_popen` with `detached=False` and a note naming the WMI error, and the refused breakaway if that also happened. Spec and handoff moved from stdin/stdout to files in `<tempdir>/pinwright-supervisor/<runId>/` (WMI gives no pipe); the spec always carries an explicit env (a WMI process gets the user's default environment) and the supervisor deletes it once read; the supervisor opens its own `supervisor.log`. `SupervisedRun` carries `detached` / `launch_mechanism` / `detach_note`; `mcp_proxy._stamp_detach` puts `detached`, `launchMechanism`, `detachNote` on `editor_run_tests`, `editor_build` and supervised `editor_start` results. Tests: WMI chosen on Windows, WMI failure -> fallback with detached False, refused breakaway named, Linux keeps setsid and never calls WMI, WMI output parsing and failures, spec file read and deleted, supervisor dying without a handoff, proxy result stamping; the real end-to-end spawns now go through WMI and assert it. Live check: supervisor pid 8636's parent is WmiPrvSE (6820), not the caller (27788); the child ran and the verdict was written. Python suite 353 OK (1 skipped). Not covered: `mode: visible` `editor_start` still spawns the editor as a direct detached child of the proxy, so it remains in the client's tree.
- `#3-visible-editor-and-tree-kill-proof` `IN-REVIEW` developer — Closed both gaps left in #2. (1) `mode: "visible"` `editor_start` on Windows now goes through `spawn_supervised(capped=False, priority="Normal", mode="visible")`, so the supervisor is WMI-created and the editor is its uncapped child (no job, so the editor outlives even the supervisor); same argv and launch switches; the result carries `detached` / `launchMechanism` / `detachNote` with the same `NOT DETACHED` fallback text. Linux keeps the direct spawn (`mcp_proxy._visible_via_supervisor()`). Checked live: a WMI-created process and its own `Popen` child both run in session 1 on `WinSta0\Default` (the interactive desktop, same session as the caller), and a window created by that grandchild with `SW_SHOWDEFAULT` is visible (`IsWindowVisible` true); the readiness wait is unchanged (endpoint probe plus `SupervisedRun.poll`, as for offscreen). Test: `test_windows_visible_goes_through_the_uncapped_supervisor_and_reports_detach`. (2) Survival proven without ending a session: `TreeKillSurvivalTest` in `tests/test_pinwright_supervisor.py` starts a stand-in caller that runs `spawn_supervised` for a sleeping child, runs `taskkill /PID <caller> /T /F`, and asserts the supervisor and child are still alive 2 s later; it then stops the child by pid and asserts the surviving supervisor wrote its verdict (`COMMAND_EXIT_NONZERO`). Counterfactual in the same class: with WMI forced off (the pre-fix direct child, `createprocess-breakaway`) the same tree kill takes down both supervisor and child. Real runs on this machine: 5 of 5 green, ~3.4 s per run of both tests, no leftover processes; kept in the suite (Windows-only). Python suite 356 OK (1 skipped). The running full suite (`gap-wave: full offscreen suite, unmasked log errors`) was not touched.
