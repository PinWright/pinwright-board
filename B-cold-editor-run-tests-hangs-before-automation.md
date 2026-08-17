---
id: B-cold-editor-run-tests-hangs-before-automation
title: "editor_run_tests against a cold content-only project hangs the editor after Python init and never reaches the automation command line — reproduced twice on the CI runner, ~0% CPU and zero I/O for 13 minutes, so package.yml's engine-install smoke can never finish"
status: OPEN
severity: High
category: bug
tags: [testing, automation, editor-lifecycle, ci, packaging, smoke, hang, cold-start]
---

# The cold `editor_run_tests` editor never starts the automation command

`editor_run_tests` launches `UnrealEditor-Cmd.exe <proj> -ExecCmds="Automation RunTests <filter>"
-TestExit=... -unattended -nopause -nosplash -nosound -RenderOffscreen -nocefaccelpaint`
(`Content/Python/mcp_proxy.py:1842-1862`) and then blocks on `proc.wait()` with no timeout
(`:1882`). Against a **cold, content-only** project on the CI runner the editor boots, initialises
the plugin, and then stops: no `LogAutomationCommandLine` line is ever written, so the automation
command the run exists to execute never starts and `-TestExit` never has anything to watch for.

`package.yml`'s engine-install smoke drives exactly this path (`ci/demo_smoke_mcp.py:399-405`,
`--cold-content-only`), so that leg cannot complete.

## Reproduction

Dispatch `package.yml` with `only: 5.8`, `run_smoke: true`. Reproduced on two consecutive runs:

| run | editor launched | last log line | outcome |
|---|---|---|---|
| [31989248279](https://github.com/PinWright/pinwright-ue-dev/actions/runs/31989248279) | 03:27 UTC | DDC maintenance, 03:34:27 | still stalled at 03:45 when the runner died |
| [31992574284](https://github.com/PinWright/pinwright-ue-dev/actions/runs/31992574284) | 04:10 UTC | DDC maintenance, 04:17:02 | still stalled at 04:31, cancelled |

Both captured logs are kept at `Plugins/PinWright/dist/runner-editor_run_tests-run*.log`.

## What the stall looks like

The log simply stops. Last three lines of the second run:

```
[2026.08.17-04.10.27:508][  0]LogPython: Python enabled via CVar 'Engine.Python.IsEnabledByDefault'
[2026.08.17-04.10.27:508][  0]LogPython: Using Python 3.11.8
[2026.08.17-04.17.02:260][  0]LogDerivedDataCache: ...: Maintenance finished in +00:04:42.169
```

That final line is written by a background thread; the game thread produced nothing after
`04:10:27`. Measured at `04:30` — 13 minutes later:

- `grep -c LogAutomationCommandLine` = **0**. The only two `LogAutomation*` lines in the whole file
  are startup `LogAutomationTest: Error: Condition failed` at `04:10:25`.
- process alive, **0 KB read, 0 KB write, 0.20 s CPU over a 15 s window** (~1.3%, i.e. the idle
  tick). Not slow — blocked.
- `ShaderCompileWorker` count 0, so it is not shader compilation.

## Ruled out

- **ZenServer.** Healthy: `LogZenServiceInstance: Local ZenServer AutoLaunch initialization
  completed in 4.918 seconds`, `LogDerivedDataCache: Display: ZenLocal: Using ZenServer HTTP
  service`. No `Failed to connect to localhost port 8558`. (This is the documented cause of a
  starved automation tick, so it was the first hypothesis.)
- **Session 0 / no desktop.** `Runner.Listener`, `UnrealEditor-Cmd` and `explorer.exe` were all in
  session **1**, so the real-RHI `-RenderOffscreen` editor had an interactive desktop.
- **A modal dialog.** Enumerated the process's top-level windows: three invisible, plus
  `Default IME`. Nothing to dismiss.
- **The plugin failing to load.** It loaded fine — `LogPinWrightSubsystem: PinWrightSubsystem
  initializing`, `PinWright integrations: loaded=[geometry,pcg] skipped=[...]`, and the run wrote a
  full `Saved/PinWright/wiki/` tree plus `gateway-port`.

## Why it was never seen before

This path has **never executed on this runner**. `abea6f86` (cold editor test lifecycle tools,
2026-07-29) landed *after* the last successful `package.yml` run (2026-07-27, run 30247642863), and
`package.yml` is manual-only. The older load smoke it replaced launched with `-ExecCmds=Quit` and no
automation at all, which is why `dist/load-smoke.log` shows a clean run.

## Consequences

- `package.yml`'s smoke leg cannot pass on this runner, so the release-package workflow cannot be
  fully verified before a Fab submission.
- It also blocks the cheap route to the observation `PROPOSED-ci-testexit.patch` is waiting on — a
  runner-session log containing `Automation Test Queue Empty <N> tests performed`. That patch stays
  deferred until then; see `ci/README.md`.
- `_editor_run_tests`' `proc.wait()` has no timeout, so on this path the verb hangs until the
  caller's own bound (the smoke passes `--timeout 7200`) rather than reporting anything. Whatever
  the root cause turns out to be, that unbounded wait is worth a bound of its own so the verb
  returns a diagnosis instead of nothing.

## Not yet investigated

Where the game thread is actually blocked. A stack sample of the wedged process (or a run with
`-stdout -FORCELOGFLUSH` and verbose `LogInit`/`LogPython`) is the obvious next step; the two
captured logs are preserved so the reproduction does not have to be re-earned.

## Related

- `B-suite-log-completeness-unverifiable` — the completion-signal family this was found under.
- `B-editor-run-tests-false-failure-on-green-run` — same verb, different failure.

## History
- `#1-cold-smoke-hangs-before-automation` `OPEN` reporter — `editor_run_tests` against a cold content-only project hangs the editor before the automation command line runs: no `LogAutomationCommandLine` line is ever emitted, the log stops after `LogPython: Using Python 3.11.8`, and the process sits at **0 KB read / 0 KB write / 0.20 s CPU over 15 s** for 13 minutes. Reproduced on two consecutive `package.yml only:5.8 run_smoke:true` dispatches (runs 31989248279 and 31992574284); logs preserved at `Plugins/PinWright/dist/runner-editor_run_tests-run*.log`. Ruled out: ZenServer (auto-launched in 4.918 s, zero port-8558 failures), session 0 (runner, editor and explorer all session 1), a modal dialog (no visible top-level windows), plugin load failure (subsystem initialised, wiki tree and `gateway-port` written). Never seen before because `abea6f86` added this path on 2026-07-29, after the last successful `package.yml` run on 2026-07-27, and that workflow is manual-only. Blocks `package.yml`'s smoke leg and, with it, the runner-session observation `PROPOSED-ci-testexit.patch` is deferred on. Secondary finding worth fixing regardless of root cause: `mcp_proxy.py:1882` waits on the editor with `proc.wait()` and no timeout, so the verb returns nothing at all rather than a diagnosis when the editor wedges.
