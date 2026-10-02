---
id: F-memory-aware-editor-launch
title: "PinWright launches editors with no machine-wide memory check or concurrency cap, and an editor killed from outside (no crash folder) is reported as a generic loss"
status: IN-REVIEW
severity: Medium
category: feature
tags: [supervisor, editor_start, editor_run_tests, editor_test_status, editor_list, memory, commit-charge, oom, external-kill, concurrency]
encounters: 1
costly: 1
lastSeen: 2026-09-29T00:00:00Z
---

# Memory-aware launching and external-kill reporting

`B-suite-run-oom-classifies-as-truncation` caps each editor's own working set (per-process Job Object
limit) and names an in-process OOM. It does not cover the machine: N editors, each under its own cap,
can still exhaust system commit together, and nothing checks free memory before a launch.

On the user's host that exhaustion is ended from outside: `C:\Tools\oom-watchdog.ps1` (user tool, not
PinWright) force-kills every `UnrealEditor`, `UnrealEditor-Cmd`, `ShaderCompileWorker`,
`UnrealBuildTool` and `ZenServer` process once commit reaches 92% or available memory drops below
3072 MB. A PinWright full suite peaks at roughly 12-18 GB, and parallel agent waves start several
editors at once. `C:\Tools\oom-watchdog.log`, 2026-09-29 (local time):

```
2026-09-29T14:34:54 kill UnrealEditor ... (8 editors, 3.1-5.1 GB each)
2026-09-29T15:06:04 kill UnrealEditor ... (5 editors) + UnrealEditor-Cmd pid=35480
2026-09-29T16:17:22 kill UnrealEditor pid=59144 5388MB
```

An editor killed this way leaves no `Saved/Crashes` folder, no fatal line, and a log that just stops,
so the run looks like a truncation or an unexplained vanish. Two earlier "silent death" reports on
this board match watchdog kills to within seconds (see `B-editor-dies-silently-cef-pie-lobby`
`#2-deaths-match-watchdog-kills`).

**Asked for:**
- Before `editor_start` / `editor_restart` / `editor_run_tests` / `editor_build` spawn anything: read
  system commit charge and available physical memory, and refuse (typed error naming the numbers and
  the running editors from `editor_list`) or queue when the launch's expected peak would push the
  machine past a configurable threshold.
- A configurable machine-wide cap on concurrently running PinWright-launched editors.
- When a supervised editor exits with no crash folder, no fatal line and no clean shutdown, report it
  as `killedExternally` (with exit code and the time the log stopped) in `editor_test_status` /
  `editor_build_status` / the supervisor result, instead of a generic lost/truncated verdict.

**Workaround:** check memory by hand before launching; run fewer editors in parallel.

## History
- `#1-watchdog-kills-parallel-editors` `OPEN` reporter - Host `X:\src\unreal\unreal-fpv` (UE 5.8): on 2026-09-29 the user's OOM watchdog killed 14 editors in two sweeps plus one more later while several agents ran editors in parallel; each killed run looked like a silent truncation. PinWright has a per-process cap but no machine-wide pre-launch check or concurrency limit, and no external-kill verdict. severity rationale: impact=Medium (soft blocker: runs lost, recoverable by relaunching fewer editors) x reach=multi-agent days on this host, no modifier -> Medium. costly=1 (runs and editor sessions lost to the sweeps).
- `#2-capacity-guard-and-external-kill` `IN-REVIEW` developer — `Content/Python/mcp_proxy.py` `Proxy._launch_capacity_guard(kind)` runs after the editor guard and before any spawn in `editor_start` / `editor_restart` (kind `editor`), `editor_run_tests` (`suite`) and `editor_build` (`build`): `EDITOR_LIMIT_REACHED` (`maxEditors`, `running`) when `$PINWRIGHT_MAX_EDITORS` (unset/0 = no cap) PinWright-launched editors (`launchedBy` != unknown, any checkout) already run, builds not counted; `LAUNCH_MEMORY_LOW` (`availableBytes`, `availableCommitBytes`, `expectedPeakBytes`, `neededBytes`, `reserveGiB`, `running`) when available physical memory (Linux `MemAvailable`) or Windows available commit (`ullAvailPageFile`) is below the expected peak (suite 16 / editor 6 / build 8 GiB, each capped at 60% of RAM) plus `$PINWRIGHT_LAUNCH_RESERVE_GB` (default 3 = the watchdog's floor; negative disables). Both texts list the running editors. Refuses, does not queue (`slot_wait` from F-multi-editor-per-checkout waits only on the checkout's port). Point-in-time check: two launches in the same minute both pass. External kill: `pinwright_supervisor.killed_externally` (Linux: exit -SIGKILL, since UE handles SIGTERM/INT/HUP gracefully and logs crashes before re-raising; Windows: non-zero exit with no `Fatal error`/`Assertion failed:`/`Caught signal` and no `RequestExit(` in the log) gives verdict `EDITOR_KILLED_EXTERNALLY` / `COMMAND_KILLED_EXTERNALLY` below TIMEOUT, MEMORY_CAP_HIT and CEF race, logs the log's last-write time; `editor_test_status` and `editor_build_status` (`build_state`) report `killedExternally` and `logStoppedAt`. The Windows rule is from source reads only (no Windows host here). Tests: new `Content/Python/tests/test_launch_capacity.py` (`LaunchCapacityGuardTest`, `LaunchCapacityWiringTest`, `KilledExternallyStatusTest`), `tests/test_pinwright_supervisor.py` `LinuxExitEvidenceTest`, `LinuxRealScopeTest.test_a_sigkill_from_outside_is_named` (real SIGKILL of the test's own child under a real scope). The other launch-test modules patch `Proxy._launch_capacity_guard` to None so they do not depend on the host's live memory. Docs: `docs/wiki-src/mcp-transport.md` (Machine-wide launch refusals, Killed from outside), `docs/test-organization.md`, `CHANGELOG.md`.
- `#3-linux-verification` `IN-REVIEW` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Python passed: `test_launch_capacity.LaunchCapacityGuardTest`, `LaunchCapacityWiringTest`, `KilledExternallyStatusTest`, and `test_pinwright_supervisor.LinuxRealScopeTest.test_a_sigkill_from_outside_is_named` (a real SIGKILL of a child under a real scope is named). The Linux half of all three asks is covered. Not demonstrated: the reporter's platform. The incident host is Windows (`C:\Tools\oom-watchdog.ps1`), and the Windows rules (available commit via `ullAvailPageFile`; an external kill = non-zero exit with no fatal line and no `RequestExit(`) come from source reads only. Needs: on Windows, a launch refused under low commit and a watchdog-killed editor reported as `killedExternally` in `editor_test_status`.
