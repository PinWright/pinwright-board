---
id: B-supervisor-linux-paths-unverified
title: "Capped supervisor and editor_list Linux paths are verified only against mocks; capSeenByEditor is always false on Linux, and a post-exit scope read can drop the OOM-kill count"
status: OPEN
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
