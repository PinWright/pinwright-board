---
id: B-version-matrix-skill-retired-launcher
title: "mcp-version-matrix skill still builds and runs suites through the deleted Content/Python/pinwright_launch.py, so its next run fails at the first other-engine build"
status: IN-REVIEW
severity: Medium
category: bug
tags: [skills, version-matrix, supervisor, editor_run_tests, editor_build, tooling, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T13:03:05Z
---

# mcp-version-matrix calls a script that no longer exists

`F-python-capped-launcher` `#7` deleted `Content/Python/pinwright_launch.py` (the file is absent from
the plugin working tree) and recorded the skill as a known deferred TODO. Nothing tracks that TODO
once `F-python-capped-launcher` reaches DONE, so this ticket does.

Still referencing the deleted CLI (plugin working tree):
- `.polyskill/skills/mcp-version-matrix/mcp-version-matrix.workflow.js:94` sets
  `cappedCli: ${clone}\Content\Python\pinwright_launch.py`; the C2 build step (`:231-235`) runs
  `& bundledPython cappedCli run ...` and the T1 suite step (`:244-248`) runs `... cappedCli suite ...`.
  Each carries `TODO(deferred)` (`:88`, `:204`, `:231`, `:244`), and the step text tells the agent to
  return FATAL when `cappedCli` is missing, so the run stops at C2 on every engine.
- `.polyskill/skills/mcp-version-matrix/SKILL.md:8` (TODO banner) and `:147` (the offscreen suite
  "runs through the retired `pinwright_launch.py suite --mode offscreen`").
- The host's installed copy (`X:\src\unreal\unreal-fpv\.claude\skills\mcp-version-matrix\`) still
  names the older `Run-Capped.ps1` / `Run-SuiteCapped.ps1` (see `F-python-capped-launcher` `#8`).

**Impact:** the version-matrix gate (release-time, multi-engine) cannot run at all: hard blocker
with no workaround inside the skill, on a rarely run path (bumped down one).
**Fix:** port C2 and T1 to the replacements: for the host engine, the proxy tools
(`editor_build` / `editor_build_status`, `editor_run_tests` / `editor_test_status`, required `mode` and
`reason`); for other engines, which the proxy tools do not target, `pinwright_supervisor.spawn_supervised`
(or a proxy option for an explicit engine path). Keep the per-process cap, BelowNormal priority,
offscreen mode and the `-ddc=InstalledNoZenLocalFallback` / GC-every / watermark switches the current
text passes. Then drop the TODO banners and regenerate the host copy.

## History
- `#1-retired-cli-still-called` `OPEN` reporter — Verified in today's review: `pinwright_launch.py` is gone, and the workflow's C2/T1 steps and `SKILL.md` still invoke it (lines above). Filed so the deferred TODO from `F-python-capped-launcher` `#7` has its own OPEN tracker.
- `#2-ported-to-supervisor-spawn` `IN-REVIEW` developer — Plugin commit `c2fc3893`. `Content/Python/pinwright_supervisor.py` gained one internal entry point for multi-engine tooling, `--spawn <request.json>` (`spawn_request`): required `reason` / `launchedBy`, required `mode` for suites, no defaults, unknown keys refused; it goes through `spawn_supervised` (0.60 RAM Job Object cap, BelowNormal, kill-on-close, WMI detach, `PROTOCOL_VERSION` spec handshake), suites use `suite_argv` (the `editor_run_tests` argv), and it prints the child and supervisor pids. The workflow's C2/T1 now write those requests, block on the supervisor pid, read the `PINWRIGHT_JOB_RESULT` / `PINWRIGHT_SUITE_RESULT` line, and grade the suite with `check_suite_log.py`; every `TODO(deferred)` and `pinwright_launch.py` reference is gone from `SKILL.md` and the workflow. The removed TODO lines also carried unescaped backticks that made the workflow a JS syntax error (plain `node --check` passes it anyway; parsing it as an async function body, as the runtime runs it, catches it). Verified: `unittest discover tests` 366 OK (1 skipped) on bundled 5.8 Python, supervisor tests on 5.3 (3.9) and 5.4 (3.11), 4 new `SpawnRequestEntryPointTest` cases, a real `--spawn` command run from Windows PowerShell 5.1 (WMI-detached, capped 37.91 GB, BelowNormal, verdict line written), and the rendered 5.3 prompt. Not verified: an actual matrix run (other-engine runs are deferred). The host's installed skill copy still needs its separate sync.
