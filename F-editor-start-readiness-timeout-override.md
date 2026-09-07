---
id: F-editor-start-readiness-timeout-override
title: "editor_start's readiness ceiling is a fixed 180 s with no per-call override, so a legitimately slow first boot returns EDITOR_START_TIMEOUT and the retry is refused as EDITOR_ALREADY_RUNNING"
status: OPEN
severity: Medium
category: feature
tags: [proxy, mcp-proxy, editor-launch, editor_start, readiness, timeout, cold-start]
encounters: 1
costly: 1
lastSeen: 2026-09-02T00:00:00Z
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
- `#1-feature-request` `OPEN` reporter — `editor_start`'s `wait="ready"` ceiling is a fixed proxy-wide 180 s (`mcp_proxy.py:974`, CLI default `:2358`) and its `inputSchema` (`:173-224`) exposes no timeout, so a caller cannot extend it for a boot known to be slow. On expiry the caller is stuck: `EDITOR_START_TIMEOUT` says "The editor may still be starting" (`:1909-1917`, `:1986`) but the port is already answering, so the retry is refused with `EDITOR_ALREADY_RUNNING … It is still starting; wait for editorReady before retrying` (`:1234-1243`) and no waiter remains. Measured on this host from `LogPinWrightSubsystem: Editor initialization completed after N seconds`: 637.48 s (`PDS-backup-2026.08.31-18.45.21.log:3277`, transport bound at 18:38:34, readiness at 18:43:39), 113.69 s (`PDS-backup-2026.09.02-08.41.10.log:3239`), 91.78 s (`PDS.log:3221`) — one run 3.5× over the ceiling and two ordinary boots at 51-63 % of it. The 637 s cause was UE's `bRunPipInstallOnStartup` fetching ~3.5 GB of torch wheels on the game thread, a host-project matter fixed in that repo and NOT filed here; it is cited only as a measurement, and a cold DDC or shader recompile reaches the same place. severity rationale: impact=Medium — a soft blocker, doable but only by polling readiness by hand through `call()` after the tool has already returned an error, i.e. many extra calls and a documented workaround; NOT High because the timeout text is honest and asserts nothing false × reach=cold-start launches, common but not every session, no modifier -> Medium. Asked for: a per-call `timeout` on `editor_start` (and `editor_restart`'s start half) defaulting to `start_timeout`; plus the already-computed `_probe_state` result (`not_ready` vs never-bound) in the timeout payload and a mention of the override in its text.
- `#2-still-open-at-upstream-head` `OPEN` reporter — Re-verified against plugin HEAD `347826a6` after the 398-commit pull from `b16f0f2b`. **Still unimplemented**; status stays `OPEN`/`Medium`, `encounters` unchanged (source re-read). Line numbers shifted, behaviour did not. The ceiling is still a single proxy-process value: `start_timeout=180.0` in the `__init__` signature (`Content/Python/mcp_proxy.py:1408`, stored `:1418` with the comment "wait=ready ceiling for editor_start") and the CLI default `parser.add_argument("--start-timeout", type=float, default=180.0)` (`:2632-2633`) — ticket cited `:974` and `:2358`. `EDITOR_START_TOOL["inputSchema"]` (`:175-231`) still exposes exactly `map`, `visible`, `wait`, `extra_args`, `unattended_script` and **no** `timeout`. The deadline is still `start + self.start_timeout` at both wait sites (`:2150`, `:2214`). The dead end is intact: the timeout payload is still `{"error": "EDITOR_START_TIMEOUT", "pid", "commandLine", "logPath"}` with the text "spawned but did not report operational readiness within %.0fs - it may still be starting" (`:2257-2270`) and carries **no** `_probe_state` field, while the retry guard still returns `EDITOR_ALREADY_RUNNING ... It is still starting; wait for editorReady before retrying` for `state in ("alive", "not_ready")` (`:1717-1729`, and the `editor_prepare_tests` copy at `:2019-2029`). So both smaller improvements asked for in `#1` are also unimplemented. Fix as filed still applies against the corrected line refs.
