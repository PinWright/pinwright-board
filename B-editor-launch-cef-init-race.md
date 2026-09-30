---
id: B-editor-launch-cef-init-race
title: "PinWright launches editors without serializing startup, and two editors started within a second race on the shared CEF webcache dir; the loser can die silently at 'Detected concurrent CEF initialization'"
status: IN-REVIEW
severity: High
category: bug
tags: [supervisor, editor_start, editor_run_tests, editor-launch, cef, webcache, startup, concurrency, check-suite-log, gap-analysis-2026-09-28]
encounters: 2
costly: 1
lastSeen: 2026-09-29T13:03:05Z
---

# Concurrent editor starts race on CEF's cache dir

The race is in the engine; PinWright's part is that its launch paths start editors back to back with
no stagger, and that a run killed by the race is not recognisable afterwards.

**Engine mechanism** (`Engine/Source/Runtime/WebBrowser/Private/WebBrowserSingleton.cpp:370-488`):
every editor picks its CEF cache dir under
`ApplicationCacheDir()/webcache` (`C:/Users/<user>/AppData/Local/UnrealEngine/<Project>/webcache_<n>_<m>`)
by checking a lockfile. Two instances that check before either creates the lockfile pick the same dir;
CEF's single-instance logic then makes `CefInitialize` fail in the second one with
`CEF_RESULT_CODE_NORMAL_EXIT_PROCESS_NOTIFIED`. On Windows the engine logs
`Detected concurrent CEF initialization for cache dir ...! Retrying... (1 of 3)`, unloads and reloads
the CEF DLLs, and retries with a new dir. The cache root has no command-line override, so a
per-instance cache dir is not available to a launcher; serializing starts is.

**Observed 2026-09-29** on `X:\src\unreal\unreal-fpv` (UE 5.8): the PinWright full suite run
`Saved/Logs/pw_gapwave_full_offscreen.log` (command line is the capped supervisor's suite argv:
`-ExecCmds="Automation RunTests PinWright,Quit" ... -RenderOffscreen -nocefaccelpaint -RunningUnattendedScript -ddc=InstalledNoZenLocalFallback`)
ends at line 2335:

```
[2026.09.29-10.10.37:810][  0]LogWebBrowser: Warning: Detected concurrent CEF initialization for cache dir C:/Users/Alexander/AppData/Local/UnrealEngine/PDS/webcache_6613_1! Retrying... (1 of 3)
```

No further line, no `Saved/Crashes` folder for that time, and the host's OOM watchdog log
(`C:\Tools\oom-watchdog.log`) shows no kill between 13:05 and 13:15 local, so the process most likely died inside the
DLL reload retry (not proven: nothing records the exit). The same warning appears in three earlier logs of this checkout
(`Automation_agentB.log:2246` 2026-09-24, `Automation_agentG_AppSumo.log:2299` 2026-09-18,
`Automation_hostowned_possession.log:2095` 2026-09-03), all on `webcache_6613_1`; those runs survived
the retry, so the race is recurring and the death is intermittent.

**Asked for:**
- Serialize editor startup in the launch paths (`editor_start`, `editor_restart`, `editor_run_tests`,
  the capped supervisor): a machine-global lock held from spawn until the new editor has passed CEF
  initialization (e.g. its log shows `LogCEFBrowser` / the webcache lockfile exists, or a fixed
  stagger of a few seconds as the minimal version). Editors launched outside PinWright still race,
  but PinWright's own parallel launches (multi-agent waves, suite plus interactive editor) stop
  racing each other.
- `check_suite_log` / `editor_test_status`: classify a log whose last line is the concurrent-CEF retry
  warning as a startup death with that named cause, instead of a generic truncation.

**Workaround:** do not start two editors within a few seconds of each other; relaunch a run that died
at this line.

## History
- `#1-suite-died-at-cef-retry` `OPEN` reporter - PinWright suite run `pw_gapwave_full_offscreen.log` (supervisor argv, UE 5.8, `X:\src\unreal\unreal-fpv`) ended at the `Detected concurrent CEF initialization ... Retrying... (1 of 3)` line with no crash folder and no watchdog kill; three earlier logs in the same checkout show the same race recovered. Mechanism by engine source read (`WebBrowserSingleton.cpp:447-488`). severity rationale: impact=editor death at startup, a lost run (Critical class, but the crash itself is the engine's and PinWright's defect is the missing launch serialization) x reach=only when two launches coincide, bumped down one -> High. costly=1 (a full suite run lost).
- `#2-exit-recorded-by-supervisor` `OPEN` reporter - Additional evidence for the same run, from the PinWright side (same lost run as `#1`, so `costly` unchanged). The exit IS recorded: the capped supervisor wrote `X:\src\unreal\unreal-fpv\Saved\Logs\pw_gapwave_full_offscreen.log.supervisor.log` (started pid 44060 at 13:10:16 local; `exit code 777003, wall 0.4 min, peak 2.32 GiB of 37.91 GiB cap, violation 0x00000000`; `last Test Started: None`). The editor exited with code 777003 about 20 s after launch, before any test, which resolves the "not proven" in `#1`. PinWright then reported it as a generic `PINWRIGHT_SUITE_RESULT verdict=EDITOR_EXIT_NONZERO ... exit=777003`: `Content/Python/pinwright_supervisor.py` `verdict_for` (`:637-643`) maps every non-zero code to `EDITOR_EXIT_NONZERO`, and neither `pinwright_supervisor.py` nor `mcp_proxy.py` has any CEF or 777003 handling (grep: only the `-nocefaccelpaint` comment at `mcp_proxy.py:1650`) or any launch stagger. Concrete shape for the two asks: (a) before `editor_start` / `editor_restart` / `editor_run_tests` spawn an editor, consult the `editor_list` census (`mcp_proxy.py` `_windows_editor_processes` / `_linux_editor_processes`) and wait until no editor of the same project started within the last few seconds; (b) in the supervisor, when the exit is non-zero, no test started, and the log's last `LogWebBrowser` line is `Detected concurrent CEF initialization`, emit a typed startup verdict (e.g. `EDITOR_STARTUP_CEF_RACE`) quoting that line, so `editor_test_status` tells the caller to relaunch instead of reporting a bare non-zero exit.
- `#3-launch-lock-and-cef-verdict` `IN-REVIEW` developer - Fixed in PinWright's launch path, no engine change (the engine offers no per-instance cache-dir switch: `ApplicationCacheDir()` + `GenerateWebCacheFolderName`, `WebBrowserSingleton.cpp:103-125,1036-1089`, only probe lockfiles). (1) Serialization: `Content/Python/mcp_proxy.py` `Proxy._acquire_launch_lock` takes a machine-wide per-user OS lock (`<tempdir>/pinwright-supervisor/editor-launch[-<uid>].lock`, `pinwright_supervisor.try_lock_file`: `flock` on Linux, `msvcrt.locking` on Windows, dropped by the OS if the holder dies) before `editor_start` / `editor_restart` / `editor_run_tests` spawn, and `_LaunchLock.release_if_settled` releases it from inside `_wait_for_ready` / `_wait_for_exit` / `_wait_for_tests_started` once the editor's (fresh) log shows it past CEF init (`pinwright_supervisor.cef_settled`: a non-CEF line after the last `LogCEFBrowser: CEF GPU acceleration` attempt with no race retry after it, or `Engine is initialized` with no attempt); otherwise at the end of the launch wait. `-NullRHI` / `-nocef` launches skip it (`launches_cef`). Queueing beyond 600 s: `EDITOR_LAUNCH_QUEUE_TIMEOUT`, nothing spawned. An `editor_start` without `-Abslog` has no log to read, so it holds the lock until ready. (2) Visibility: `EDITOR_STARTUP_CEF_RACE` (with `exitCode`, `cefLine`) replaces `EDITOR_EXITED_BEFORE_TESTS` / `EDITOR_EXITED_BEFORE_READY` when the exited editor's last CEF log line is the retry warning (`pinwright_supervisor.cef_race_line`); the supervisor's suite verdict becomes `EDITOR_STARTUP_CEF_RACE` (`verdict_for(..., cef_race)`, `_scan_log` `cefRaceLine`, only when no test started); `check_suite_log` / `editor_test_status` keep `DID_NOT_COMPLETE` but the reason names the CEF startup death and says relaunch (`parse_automation_log` `cefRaceLine`, `classify_run_evidence`). Docs: `docs/wiki-src/mcp-transport.md` lifecycle bullets + test-run error table + checker state row; plugin `CLAUDE.md` startup-failure list. Tests: `Content/Python/tests/test_launch_lock_and_build_lease.py` `FileLockTest` (incl. a real child process holding the lock, released when it is killed), `CefLogClassificationTest`, `LaunchLockTest`; all 19 tests of the module fail against HEAD's Python. `test_mcp_proxy_editor_start.py` now points the lock at a temp path for its module. Not proven live: two real editors launched at once on Windows (needs a manager-run pair of `editor_run_tests` calls; expect the second to start ~CEF-init later and both to reach tests).
