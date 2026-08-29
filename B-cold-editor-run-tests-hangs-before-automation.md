---
id: B-cold-editor-run-tests-hangs-before-automation
title: "editor_run_tests against a cold content-only project hangs the editor after Python init and never reaches the automation command line — reproduced twice on the CI runner, ~0% CPU and zero I/O for 13 minutes, so package.yml's engine-install smoke can never finish"
status: IN-REVIEW
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
- `#2-additional-watchdog-split` `OPEN` reviewer-a — Additional evidence: **Adversarial review A — partially current.** Actuality: PARTIAL. Framing: the reported pre-automation stall is not independently verifiable here, while the ticket also conflates it with a confirmed unbounded wrapper wait; its `-TestExit` command description is stale because current `_editor_run_tests` launches `Automation RunTests ...; Quit`. Proposed fix: INCOMPLETE, a bounded wait is necessary containment but cannot repair the unknown editor/game-thread stall, and adding `-TestExit` alone is a band-aid without a defined report/terminal-marker contract. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:1763`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:1792`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\tests\test_mcp_proxy_editor_start.py:1741-1751`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\.github\workflows\package.yml:231-233`. Runtime: NOT VERIFIED. Recommendation: REFRAME; split the confirmed watchdog/diagnostics defect from the cold-editor root-cause investigation, then collect a fresh stack-sample repro before choosing the engine fix.
- `#3-additional-verify-run-and-fix` `OPEN` reporter — Additional evidence: **Adversarial review B — agrees with A that containment and root-cause work must be split, but corrects the invocation and repro history.** Actuality: PARTIAL. Framing: the live GitHub records do not support “reproduced twice”: run `31989248279`'s UE 5.8 job `95269641550` failed in **Build release package** before the smoke step (no editor was launched), while run `31992574284`'s UE 5.8 job `95278606034` built, entered the smoke at 04:10, and was cancelled while the cold proxy call waited. Also, the reported run's head `e9dafb3a` already contained `3151e26a`'s `-TestExit=Automation Test Queue Empty`; A's “stale `-TestExit`” disagreement is true only for this older local checkout (`mcp_proxy.py:1763`), not for that reproduction. The local checkout still has the confirmed unbounded `proc.wait()` (`:1792`), but later history `b4dfab2f` (bounded wait and PID-scoped sample/cleanup), `a01910a3` (idle startup watchdog), and `c35348dd` (startup modal suppression) is a systemic response, not a bare timeout or flag. Proposed fix: INCOMPLETE in the ticket as written; the later implementation is SYSTEMIC, but its packaged runtime behavior has not been verified. Evidence: `X:\src\unreal\.pinwright-board\B-cold-editor-run-tests-hangs-before-automation.md:24-31`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:1760-1792`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\.github\workflows\package.yml:226-233`, `https://github.com/PinWright/pinwright-ue-dev/commit/3151e26a`, `https://github.com/PinWright/pinwright-ue-dev/commit/b4dfab2f`, `https://github.com/PinWright/pinwright-ue-dev/commit/a01910a3`. Runtime: NOT VERIFIED. Recommendation: VERIFY FIRST; package the current target commit and confirm the smoke returns a structured startup-stall/timeout diagnosis, PID-scoped cleanup, and no indefinite wait, then close/split the remaining engine root-cause investigation.
- `#4-mechanism-named-and-regrowth-gated` `IN-REVIEW` developer — **Root cause named, with engine evidence re-derived in `X:\src\unreal\unreal-fpv-new` against `C:\UE_5.8\Engine\Source`** (reviews #2/#3 cited `unreal-fpv-dev`, a different checkout). The mechanism is a Slate modal raised during startup under `-unattended` WITHOUT `-RunningUnattendedScript`. `FSlateApplication::AddModalWindow` gates on `GIsRunningUnattendedScript` and nothing else (`Runtime/Slate/Private/Framework/Application/SlateApplication.cpp:2134`; grepping `IsUnattended` across that whole file returns **zero** hits), and only `-RunningUnattendedScript` sets that global (`Runtime/Launch/Private/LaunchEngineLoop.cpp:6857-6860`, inside `#if !UE_BUILD_SHIPPING`). `-unattended` sets `FApp::IsUnattended()`, which that path never reads, so it suppresses only the native `MessageBoxExt` prompts. All four citations verified line-for-line on this host. **This explains the ticket's one wrong turn:** "a modal dialog — enumerated the top-level windows, three invisible plus `Default IME`, nothing to dismiss" is exactly what a Slate modal under `-RenderOffscreen` looks like, so that observation supports the hypothesis instead of refuting it. **The reported hang was already fixed before this pass, and #3 was right that the response was systemic.** `c35348dd` (2026-08-20) added `-RunningUnattendedScript` to the test-run launch, carrying a measurement that matches this ticket's signature (DDC maintenance finished normally, then zero tests for 25 minutes at 0.8% CPU with no `ShaderCompileWorker` alive); the 2026-08-17 reproductions predate it. `0db49088` (2026-08-23) then deleted `_editor_run_tests` outright, replacing it with `editor_prepare_tests`, which returns `COMMAND_READY` carrying `launch.argv` + `checker.argv` and spawns nothing — so the unbounded `proc.wait()` of consequence #3 went with the verb that owned it. Current `launch_argv` carries both `-RunningUnattendedScript` and `-nocefaccelpaint`; `ci/demo_smoke_mcp.py:227-238` already asserts both on the returned command, and the `--cold-content-only` path no longer launches any editor through the test verb at all. **Ruled out, with reasons.** `-nocefaccelpaint` (the ticket's leading candidate) was already on the reproduced launch and is on every launch in the tree — not the cause. `-AutoDeclinePackageRecovery` missing from `launch_argv` is a non-defect: `FPackageAutoSaver::OfferToRestorePackages` already guards on `!FApp::IsUnattended()` (`Editor/UnrealEd/Private/PackageAutoSaver.cpp:683`), so that prompt cannot fire under `-unattended` at all. The preserved logs `dist/runner-editor_run_tests-run*.log` are **absent from this checkout** (`dist/` holds only `_stage`, `fab-listing.md`, `verify-blank-capture-20260813`), so no log-side re-derivation was possible. **Landed here — one live instance plus a regrowth gate.** `ci/demo-smoke.ps1:139` was the last launcher in the tree still passing `-unattended` alone, and it launches the coldest host in CI (a blank content-only `.uproject` written two steps above it); added `-RunningUnattendedScript` with the citation. New static gate `Content/Python/check_unattended_flags.py` — stdlib-only, no editor, in the style of `check_test_ids.py` — scans `.ps1`/`.yml`/`.yaml`/`.js` for an argv-shaped `-unattended` lacking the pair switch (at least 2 companion launch switches within a ±6-line window, so wrapping style is irrelevant, and the runner's `--unattended` config flag is excluded by lookbehind). `.md`/`.py` are out of scope by construction: both are full of prose naming `-unattended` while explaining this defect, and an allowlist that large is what gets a gate disabled rather than fixed. Self-test `Content/Python/tests/test_unattended_flag_scan.py` (9 tests, counterfactual included) and CI job `unattended-flag-scan` in `.github/workflows/ci.yml`, self-test-before-scan like its two siblings. Documented in `Docs/test-organization.md`. **Verified without an editor or a build, per the wave rules:** the scanner over the tree returns `CLEAN` across 10 command files and printed exactly one `UNATTENDED-ALONE` before the fix; `python -m unittest discover tests` = **214 tests OK** on UE 5.8's bundled 3.11.8. **Two items deliberately not fixed.** (1) `_wait_for_exit` (`mcp_proxy.py:2353`) still calls `proc.wait()` with no cutoff — reachable only from `editor_start {wait:"exit"}`, unbounded since `abea6f86`, and the plugin `CLAUDE.md` records proxy-owned test timeouts as a *retired* contract, so bounding it is a separate decision about a different verb, not this ticket's fix. (2) `.polyskill/skills/mcp-version-matrix/SKILL.md:133-134` summarises its launcher without the pair switch while the `mcp-version-matrix.workflow.js:213` it defers to has it — doc drift the scan cannot see by design, left for the skill owner because `.polyskill` sources need an `npx polyskill` rebuild to reach any runtime. **What a reviewer should check:** that the four engine citations hold on their host, that `unattended-flag-scan` is green in a real CI run, and that a `package.yml` dispatch now completes its smoke leg — the last is the only claim here that no amount of source reading can settle.
