---
id: F-editor-start-readiness-timeout-override
title: "editor_start's readiness ceiling is a fixed 180 s with no per-call override, so a legitimately slow first boot returns EDITOR_START_TIMEOUT and the retry is refused as EDITOR_ALREADY_RUNNING"
status: DONE
severity: Medium
category: feature
tags: [proxy, mcp-proxy, editor-launch, editor_start, readiness, timeout, cold-start]
encounters: 3
costly: 1
lastSeen: 2026-09-28T09:32:00Z
---

# `editor_start` has no per-call readiness timeout

`editor_start`'s `wait="ready"` ceiling is `start_timeout`, a proxy-process value
fixed at **180 s** (`Content/Python/mcp_proxy.py:974`, CLI default at `:2358`).
The tool's `inputSchema` (`:173-224`) exposes `map`, `visible`, `wait`,
`extra_args` and `unattended_script` — **no timeout**. A caller who knows this
boot will be slow has no way to say so.

On expiry the proxy returns `EDITOR_START_TIMEOUT` with the honest caveat "The
editor may still be starting" (`:1909-1917`, `:1986`), and then the situation is a
dead end inside the tool: the editor **is** answering the port, so a retry hits
the live-endpoint guard and comes back `EDITOR_ALREADY_RUNNING … It is still
starting; wait for editorReady before retrying` (`:1234-1243`). There is nothing
to wait *with* — `editor_start` is the waiter, and it has already given up.

## Measured on this host

| Run | Init time | vs. 180 s ceiling |
|-----|-----------|-------------------|
| 2026-08-31, `Saved/Logs/PDS-backup-2026.08.31-18.45.21.log:3277` | **637.48 s** | 3.5× over |
| 2026-09-02, `Saved/Logs/PDS-backup-2026.09.02-08.41.10.log:3239` | 113.69 s | 63 % of it |
| 2026-09-02, `Saved/Logs/PDS.log:3221` | 91.78 s | 51 % of it |

(`LogPinWrightSubsystem: Editor initialization completed after N seconds`.)

The 637 s run was UE's own `bRunPipInstallOnStartup` pulling ~3.5 GB of
torch/torchvision/torchaudio wheels synchronously on the game thread — a **project**
matter, fixed in the host repo, and deliberately not filed here. It is quoted only
because it is a clean measurement of the failure mode: the transport bound the port
at `18:38:34` and readiness landed at `18:43:39`, five minutes later, so the port
answered for the whole window in which `editor_start` timed out.

Nothing about that cause is special. A cold DDC, a shader recompile, a first boot
after an engine upgrade, or any plugin doing synchronous startup work puts a real
project over 180 s. And the two ordinary boots above already sit at half to
two-thirds of the ceiling, so the margin on this project is thin without any
pathology at all.

## What is asked for

A `timeout` (seconds) property on `editor_start`'s `inputSchema`, defaulting to the
current `start_timeout` and overriding it for that call. `editor_restart` should
take the same knob for its start half.

Two smaller improvements worth folding in, both cheap because the state already
exists — `_probe_state` distinguishes `not_ready` from `alive` and the
already-running text uses it:

- Report the observed state in the `EDITOR_START_TIMEOUT` payload
  (`not_ready` vs never-bound), so "still booting" is machine-readable rather than
  a sentence.
- Make the timeout text name the override, the way `_startup_modal_hint` names
  `unattended_script`.

## See also
- `F-editor-readiness-probe` (OPEN) — the readiness *signal*; parts of it have since
  shipped as `_probe_state`'s `not_ready`. This ticket is about the *ceiling* on
  waiting for that signal, and about there being no waiter to resume on.
- `B-editor-start-blocks-on-zenserver-modal` (IN-REVIEW) — the other way a boot
  reaches the same 180 s timeout; fixed by failing fast on the modal, which does not
  help a boot that is genuinely still working.

## History
- `#3-additional-timeout-on-healthy-boot` `OPEN` orchestrator — Third observation, 2026-09-07T08:30Z, EAContentExamples58 on UE 5.8. `editor_start {unattended_script:true}` after a render-thread crash returned `Editor (pid 9720) spawned but did not report operational readiness within 180s - it may still be starting`; the process was healthy and port 27145 was listening about 20 s later (`netstat` at 08:33:04Z, log writing the asset-registry cache at 08:33:02Z). The tool reports a failure the caller must then disprove by hand, and every restart in a multi-agent session pays that check. Cheap this time (one port probe), so `costly` is unchanged and no cost bump; `encounters` 1 -> 2 and `lastSeen` refreshed. The ask stands: a `readiness_timeout` argument, or the tool continuing to wait while the process is alive and the log is advancing.
- `#1-feature-request` `OPEN` reporter — `editor_start`'s `wait="ready"` ceiling is a fixed proxy-wide 180 s (`mcp_proxy.py:974`, CLI default `:2358`) and its `inputSchema` (`:173-224`) exposes no timeout, so a caller cannot extend it for a boot known to be slow. On expiry the caller is stuck: `EDITOR_START_TIMEOUT` says "The editor may still be starting" (`:1909-1917`, `:1986`) but the port is already answering, so the retry is refused with `EDITOR_ALREADY_RUNNING … It is still starting; wait for editorReady before retrying` (`:1234-1243`) and no waiter remains. Measured on this host from `LogPinWrightSubsystem: Editor initialization completed after N seconds`: 637.48 s (`PDS-backup-2026.08.31-18.45.21.log:3277`, transport bound at 18:38:34, readiness at 18:43:39), 113.69 s (`PDS-backup-2026.09.02-08.41.10.log:3239`), 91.78 s (`PDS.log:3221`) — one run 3.5× over the ceiling and two ordinary boots at 51-63 % of it. The 637 s cause was UE's `bRunPipInstallOnStartup` fetching ~3.5 GB of torch wheels on the game thread, a host-project matter fixed in that repo and NOT filed here; it is cited only as a measurement, and a cold DDC or shader recompile reaches the same place. severity rationale: impact=Medium — a soft blocker, doable but only by polling readiness by hand through `call()` after the tool has already returned an error, i.e. many extra calls and a documented workaround; NOT High because the timeout text is honest and asserts nothing false × reach=cold-start launches, common but not every session, no modifier -> Medium. Asked for: a per-call `timeout` on `editor_start` (and `editor_restart`'s start half) defaulting to `start_timeout`; plus the already-computed `_probe_state` result (`not_ready` vs never-bound) in the timeout payload and a mention of the override in its text.
- `#2-still-open-at-upstream-head` `OPEN` reporter — Re-verified against plugin HEAD `347826a6` after the 398-commit pull from `b16f0f2b`. **Still unimplemented**; status stays `OPEN`/`Medium`, `encounters` unchanged (source re-read). Line numbers shifted, behaviour did not. The ceiling is still a single proxy-process value: `start_timeout=180.0` in the `__init__` signature (`Content/Python/mcp_proxy.py:1408`, stored `:1418` with the comment "wait=ready ceiling for editor_start") and the CLI default `parser.add_argument("--start-timeout", type=float, default=180.0)` (`:2632-2633`) — ticket cited `:974` and `:2358`. `EDITOR_START_TOOL["inputSchema"]` (`:175-231`) still exposes exactly `map`, `visible`, `wait`, `extra_args`, `unattended_script` and **no** `timeout`. The deadline is still `start + self.start_timeout` at both wait sites (`:2150`, `:2214`). The dead end is intact: the timeout payload is still `{"error": "EDITOR_START_TIMEOUT", "pid", "commandLine", "logPath"}` with the text "spawned but did not report operational readiness within %.0fs - it may still be starting" (`:2257-2270`) and carries **no** `_probe_state` field, while the retry guard still returns `EDITOR_ALREADY_RUNNING ... It is still starting; wait for editorReady before retrying` for `state in ("alive", "not_ready")` (`:1717-1729`, and the `editor_prepare_tests` copy at `:2019-2029`). So both smaller improvements asked for in `#1` are also unimplemented. Fix as filed still applies against the corrected line refs.
- `#4-timeout-under-machine-load` `OPEN` reporter - Fourth observation, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`. `editor_start` returned `EDITOR_START_TIMEOUT` at 180 s while the shared machine was heavily loaded (load average about 25 on the 5-minute figure at filing); the editor became ready shortly afterwards with no intervention. `Saved/Logs/PDS.log:3380`: `Editor initialization completed after 163.67 seconds`, so init alone took 91 % of the ceiling; the two earlier boots today took 135.69 s (`PDS-backup-2026.09.28-08.51.29.log:3375`) and, on 2026-09-25, 82.02 s. At plugin `8fcc0b2a` the ceiling is still only the proxy CLI flag `--start-timeout` (`Content/Python/mcp_proxy.py:2962`, stored `:1686`, used at `:2457` and `:2521`), `.mcp.json` here does not set it, and the tool schema still has no timeout. The timeout text already says "it may still be starting", but the payload is still `isError` with no probe state, and a retry still hits `EDITOR_ALREADY_RUNNING` (`:1986-1998`). Cheap (the editor came up on its own), so `costly` unchanged.
- `#5-per-call-timeout` `IN-REVIEW` developer - `editor_start` and `editor_restart` take `timeout` (number, seconds, above 0 and at most `READY_TIMEOUT_MAX` = 3600), the `wait: "ready"` ceiling for that call; absent, the proxy's `--start-timeout` (180) still applies. Validated by `Proxy._ready_timeout` before the slot wait / guard / spawn (`INVALID_ARGUMENTS`, `param: "timeout"`, also for a bool, a string, 0, >3600, or `timeout` with `wait: "exit"`, which has no cutoff); `editor_restart` validates it before stopping anything and forwards it to the start half. `_wait_for_ready` takes `timeout=`. `EDITOR_START_TIMEOUT` now carries `probeState` (`not_ready` = port answers, still initializing; `unresponsive`; `not_running` = nothing bound the port) and `timeoutSeconds`, and its text names the state, says the editor was left running, and names the `timeout` override. Files: `Content/Python/mcp_proxy.py` (both schemas, `READY_TIMEOUT_MAX`, `_ready_timeout`, `_editor_start`, `_editor_restart`, `_wait_for_ready`), `Content/Python/tests/test_mcp_proxy_editor_start.py`, `docs/wiki-src/mcp-transport.md` (tool table + `timeout` bullet), `CHANGELOG.md`. Tests (engine python unittest): `ProxyEditorStartTest.test_per_call_timeout_overrides_the_proxy_ceiling_and_reports_not_ready` (fails if the override is ignored: proxy ceiling 5 s, call 0.01 s), `.test_default_timeout_is_the_proxy_ceiling`, `.test_invalid_timeout_is_refused_before_anything_spawns`, `.test_timeout_with_wait_exit_is_refused`, `.test_timeout_is_advertised_on_start_and_restart`, `.test_direct_ready_timeout_reports_abslog_path` (now asserts `probeState: not_running`), `ProxyEditorRestartTest.test_timeout_is_forwarded_to_the_start_half`, `.test_invalid_timeout_is_refused_before_the_quit`. Full `unittest discover tests`: 435 ran, all pass, 5 skipped. Not done: the `#3` alternative (keep waiting while the log advances) - the explicit override is the ask in `#1`.
- `#6-review-fixes` `IN-REVIEW` developer - Review follow-up. (1) GPU-crashed editors fail fast: `_probe_state` keeps returning `unresponsive` for a `gpuCrashed` ping but now types the detail as `_GpuCrashedDetail` (a `str` subclass), and `_wait_for_ready` returns `EDITOR_UNRESPONSIVE` with that diagnostic on the first such probe instead of polling a dead editor up to the (now up to 3600 s) ceiling. (2) The `EDITOR_START_TIMEOUT` text is per `probeState`: `not_ready` says editor_start is refused `EDITOR_ALREADY_RUNNING` until ready and to poll via `call()`/`editor_list`; `unresponsive` quotes the probe's own diagnostic (no longer the false "did not answer the liveness ping"); `not_running` says the pid is still booting and NOT to call editor_start again (the guard would allow a second launch - filed as `B-editor-start-guard-ignores-booting-editor-process`). The `timeout` advice is now "on the next launch". (3) `editor_restart`'s refusal of a non-alive editor passes the probe detail. `docs/wiki-src/mcp-transport.md` `timeout` bullet reworded to match. New tests in `test_mcp_proxy_editor_start.py`: `ProxyEditorStartTest.test_gpu_crashed_editor_fails_fast_instead_of_polling_to_the_timeout`, `.test_unbound_port_timeout_warns_against_a_second_launch`, `.test_unresponsive_timeout_quotes_the_probe_diagnostic`, `ProxyEditorRestartTest.test_wedged_editor_refusal_carries_the_probe_diagnostic`. Full `unittest discover tests`: 462 ran, OK, 5 skipped.
- `#7-verified-linux` `DONE` tester — Fix commit 123efe87. Python run3: 462 tests OK (5 skipped, none of them these). This includes `ProxyEditorStartTest.test_per_call_timeout_overrides_the_proxy_ceiling_and_reports_not_ready`, `.test_default_timeout_is_the_proxy_ceiling`, `.test_invalid_timeout_is_refused_before_anything_spawns`, `.test_timeout_with_wait_exit_is_refused`, `.test_timeout_is_advertised_on_start_and_restart` and `.test_direct_ready_timeout_reports_abslog_path`. It also includes `.test_gpu_crashed_editor_fails_fast_instead_of_polling_to_the_timeout`, `.test_unbound_port_timeout_warns_against_a_second_launch` and `.test_unresponsive_timeout_quotes_the_probe_diagnostic`, plus `ProxyEditorRestartTest.test_timeout_is_forwarded_to_the_start_half`, `.test_invalid_timeout_is_refused_before_the_quit` and `.test_wedged_editor_refusal_carries_the_probe_diagnostic`. Acceptance: a per-call `timeout` is on both `editor_start` and `editor_restart`'s start half and defaults to `--start-timeout`; `EDITOR_START_TIMEOUT` carries `probeState` (`not_ready` / `unresponsive` / `not_running`) and `timeoutSeconds`; the text names the override. Coverage limit: unit tests with a mocked editor, with no live slow boot. The #3 alternative (keep waiting while the log advances) was not done; the explicit override asked for in #1 is.
