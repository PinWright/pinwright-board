---
id: F-memory-aware-editor-launch
title: "PinWright launches editors with no machine-wide memory check or concurrency cap, and an editor killed from outside (no crash folder) is reported as a generic loss"
status: OPEN
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
