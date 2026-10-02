---
id: B-supervisor-linux-paths-unverified
title: "Capped supervisor and editor_list Linux paths are verified only against mocks; capSeenByEditor is always false on Linux, and a post-exit scope read can drop the OOM-kill count"
status: IN-REVIEW
severity: Medium
category: bug
tags: [supervisor, linux, systemd-run, cgroup, memory-cap, editor_list, editor_run_tests, capSeenByEditor, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T13:03:05Z
---

# Linux supervisor paths: never run on a real host

`Content/Python/pinwright_supervisor.py` has a full Linux branch (docstring `:49-64`): containment via
`systemd-run --user --scope --quiet -p MemoryMax=<bytes> -- argv` (`systemd_scope_prefix`, `:517-531`),
scope discovery by reading `/proc/<pid>/cgroup` and the scope's `memory.max` (`linux_wait_for_scope`,
`:554-568`), peak and OOM kills from `memory.peak` / `memory.events` (`linux_scope_usage`, `:571-580`),
nice + `PR_SET_PDEATHSIG` + own process group (`linux_preexec`, `:501-514`), and `killpg` on timeout
and exit. `mcp_proxy.py` `_linux_editor_processes` (`:1950-1977`) feeds `editor_list`. The only
coverage is `Content/Python/tests/test_pinwright_supervisor.py` (`LinuxContainmentTest`, `:234-286`),
which runs these branches against a fake `/proc` and cgroup tree with process creation and libc mocked;
the file header (`:7`) says so, and `F-python-capped-launcher` `#4` records "Linux paths are
unit-tested with mocks only". Unverified on a real host:

1. `systemd-run --user --scope` keeps the editor on the Popen PID (the scope execs the command), so
   `linux_wait_for_scope(proc.pid, ...)` finds the right cgroup.
2. The memory controller is delegated to the user manager, so `memory.max` reads back; on hosts where
   it is not, the run silently goes uncapped with the stated reason (distro-dependent).
3. `memory.peak` exists only on kernel 5.19+; older kernels report peak 0, leaving cap detection to
   `oom_kill` and log lines.
4. `nice`, `PR_SET_PDEATHSIG` surviving the `systemd-run` exec, and `killpg` reaching every editor
   child (ShaderCompileWorker).

Two defects visible by code read:
- **`capSeenByEditor` is always false on Linux.** `_CAP_SEEN` (`:586-587`) matches only the two
  Windows `LogMemory` lines (`WindowsPlatformMemory.cpp:115`, `:479`); the Unix platform layer does not
  read cgroup limits (no `cgroup` / `memory.max` reference under
  `C:\UE_5.8\Engine\Source\Runtime\Core\Private\Unix`). The result line then reports
  `capSeenByEditor=False` on every capped Linux run, which reads as a failed cap.
- **The final scope read can erase the OOM-kill count.** After the child exits, `supervise`
  (`:824-826`) re-reads `linux_scope_usage(child.scope)` and assigns its `oom_kills` over the polled (`:808`)
  value. If systemd has already removed the transient scope's cgroup (it stops an empty scope), both
  files are unreadable, `oom_kills` becomes 0, and `violation_is_memory` is false for a run the kernel
  OOM-killed. The in-loop poll runs every 2 s and returns as soon as the process dies, so it usually
  has not seen the kill yet.

**Impact:** on Linux a memory-capped run can be misreported (no cap seen, OOM kill classified as a
plain non-zero exit). Linux is a supported secondary platform; reach is low.
**Fix:** run `editor_run_tests` (capped, offscreen and headless) and `editor_list` once on a real
Linux host (the Linux dev box or CI build machine) and record the supervisor log. In code: report
`capSeenByEditor` as not applicable on Linux (or detect the cap from the editor's `Memory total` line
against the scope limit), and keep the maximum of the polled and final `oom_kill` counts instead of
overwriting it.

## History
- `#1-linux-paths-mock-only` `OPEN` reporter — Found in today's review of the supervisor. Both code defects are from source reads (supervisor lines above; engine `WindowsPlatformMemory.cpp:115`, `:479`); nothing was run on Linux.
- `#2-verified-on-linux-host` `IN-REVIEW` developer — Ran the Linux branches on the real Linux dev host (Ubuntu 22.04.5, systemd 249, kernel 6.8, memory+pids delegated to user@1000): (1) `systemd-run --user --scope` keeps the Popen pid (the scope execs), and the child's `/proc/self/cgroup` is the `run-r*.scope` `linux_wait_for_scope` returns; (2) `memory.max` reads back (every suite today: `cap 75.51 GiB`); (3) `memory.peak` exists (kernel 6.8; suites report peaks 4.9-13.6 GiB); (4) the child runs at nice 10 as its own process-group leader, `PR_GET_PDEATHSIG` reads 9 after the exec, and `killpg` reaped a `sleep` grandchild. `editor_list`'s `_linux_editor_processes` found a renamed process under a real capped scope by `/proc/<pid>/exe` and skipped three zombie `UnrealEditor` entries (left unreaped by the wt1 checkout's proxy). Defects confirmed and fixed in `Content/Python/pinwright_supervisor.py`: **the post-exit read did erase the OOM kill** — reproduced: a child OOM-killed inside the first 2 s poll left polled `oom_kill=0`, and its scope was already removed (reads ENODEV) at the final read, so the run read as a plain exit -9; the final read now keeps the max, and when the scope is gone and the exit is SIGKILL it takes the delta of the parent slice's hierarchical `memory.events` `oom_kill` captured at scope discovery. **`capSeenByEditor` is `n/a` on Linux** (UE's Unix layer logs only `Physical RAM available (not considering process quota)`); the supervisor log names the scope whose `memory.max` read back. Two more Linux defects found on the host and fixed: **every drained Linux suite was `verdict=EDITOR_EXIT_NONZERO exit=1`** because Unix `RequestExit(true)` from `-TestExit` is `_exit(1)` (`UnixPlatformMisc.cpp:345-355`) — exit 1 whose last `RequestExit(` line is `FEngineLoop::Tick.GScopedTestExit` now reads `EDITOR_EXITED` (exit stays 1 in the line); and **systemd 249's user manager leaked transient scopes** (45 units `active (running)` with 0 tasks on this host: probes running `true`, unit-test children, two finished editors and a build) — the probe now runs as a named `pinwright-probe-<hex>.scope` and is stopped, and a scope still present after the final read is stopped (`systemctl --user stop --no-block`). Tests: `tests/test_pinwright_supervisor.py` `LinuxRealScopeTest` (real systemd scope, no mocks; skipped where none is available: `test_scope_holds_the_popen_pid_tree_niced_grouped_and_reaped`, `test_an_oom_kill_is_counted_although_the_scope_is_gone` — fails with the fallback removed, `test_a_sigkill_from_outside_is_named`) and `LinuxExitEvidenceTest` (`test_forced_test_exit_of_a_drained_linux_run_is_not_a_failure`, `test_cap_seen_is_written_as_na`, ...); `LinuxContainmentTest.test_systemd_prefix_and_fallbacks` updated for the named probe. Docs: `docs/wiki-src/mcp-transport.md` (Killed from outside; Platform differences memory-cap row), `docs/test-organization.md`. Not run here: a deliberately capped real-editor OOM (no editor may be started from this session); the existing 45 leaked units were left in place (`systemctl --user stop` on each empty `run-r*.scope` clears them).
- `#3-linux-verification` `IN-REVIEW` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Python passed, including `LinuxRealScopeTest` against real systemd user scopes on this host with no skip (`test_scope_holds_the_popen_pid_tree_niced_grouped_and_reaped`, `test_an_oom_kill_is_counted_although_the_scope_is_gone`, `test_a_sigkill_from_outside_is_named`) and `LinuxExitEvidenceTest` (`test_cap_seen_is_written_as_na`, `test_forced_test_exit_of_a_drained_linux_run_is_not_a_failure`, ...). Items 1-4 and both code defects are covered at that level. Not demonstrated: the fixed supervisor on a real editor suite, which the Fix asks for. The w23 runs were launched directly (no supervisor log), and the newest supervisor logs under `Saved/PinWright/test-runs` (e.g. `636bc41c`) predate the fix and still read `verdict=EDITOR_EXIT_NONZERO ... capSeenByEditor=False`. No capped headless run and no capped real-editor OOM were run. Needs: `editor_run_tests` offscreen and headless through a restarted proxy, with result lines reading `EDITOR_EXITED` and `capSeenByEditor=n/a`.
