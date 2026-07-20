---
id: F-proxy-editor-start-rpc
title: "mcp_proxy.py: add an editor-start RPC (visible or invisible run) with readiness-gated return"
status: IN-REVIEW
severity: Medium
category: feature
tags: [proxy, mcp-proxy, editor-launch, startup, readiness, headless]
encounters: 1
lastSeen: 2026-07-17T13:30:00+03:00
---

# mcp_proxy.py: add an editor-start RPC (visible or invisible run)

Maintainer-requested (2026-07-17). Agents currently launch the editor
out-of-band (`Start-Process UnrealEditor.exe <project> -unattended -nosplash`),
then hand-roll a port-poll loop plus a blind sleep for python init (see
`F-editor-readiness-probe`). No in-editor RPC can ever cover this — the editor
process doesn't exist yet — so the capability belongs in the stdio python proxy
(`Plugins/PinWright/Content/Python/mcp_proxy.py`), which lives outside the
editor process and already mediates client↔editor traffic.

**Requested shape:**
- A proxy-side RPC/tool that starts the editor for the project, with an option
  for **visible or invisible** runs (normal window vs windowless/offscreen — the
  project's established headless flags are `-unattended -nosplash`, plus
  `-RenderOffScreen` for fully windowless runs).
- The call should not return "started" on mere process spawn: it should gate on
  actual readiness (poll the HTTP port, then the cheap C++ readiness state from
  `F-editor-readiness-probe`), holding the request open in the proxy — per the
  same maintainer ruling, blocking waits live in the proxy python, not in C++.
- Useful extras: return the OS PID + resolved command line (mirror the shape of
  in-editor `editor.launch_standalone`, which does this for `-game` processes),
  and refuse/report if an editor for this project is already running.

**Related:** `F-editor-readiness-probe` (the readiness state this RPC gates on;
same proxy-owns-the-wait ruling), `B-proxy-blocks-minutes-on-hung-editor` (the
proxy's liveness probe — a start RPC must not confuse "starting" with "hung"),
`editor.launch_standalone` (in-editor precedent for spawning processes with a
visible/invisible distinction, but it cannot start the editor itself).

## History
- `#1-maintainer-request` `OPEN` maintainer — Requested alongside the readiness-probe ruling: "maybe add editor start rpc to py, with option for visible or invisible runs." Filed with the readiness-gated-return shape; blocking lives in mcp_proxy.py per the same ruling.
- `#2-editor-start-tool` `IN-REVIEW` developer — Added a dedicated proxy-local `editor_start` MCP tool in `Content/Python/mcp_proxy.py` (always advertised in `tools/list` alongside `call`; routed locally before the forward path). Args `{visible, wait, extra_args}`. Two `wait` modes: `ready` (default) blocks until the editor answers `ping` and leaves it running (returns pid/url/port/elapsed, bounded by `--start-timeout` 180s, reports already-running instead of double-spawning); `exit` blocks until the spawned process terminates (returns exitCode/duration/logPath, bounded by `--run-timeout` 1800s) for `-ExecCmds=...;Quit` test runs / commandlets. `visible=false` uses the repo's windowless set (`-RenderOffScreen -unattended -nopause -nosplash -nocefaccelpaint`, hidden window; never `-NullRHI`). Editor binary auto-detected (default: walk up from the bundled `sys.executable` to the engine tree; overrides `--editor-exe` > `$UE_ROOT` > `.uproject` EngineAssociation). Spawns detached; never kills a slow child (starting != hung). Stdlib-only. Discoverability hints added to the dead-editor error strings. New unit tests `tests/test_mcp_proxy_editor_start.py` (pure command-builder + exe-resolver + `_abslog_path`); full suite 29 tests green under the bundled Python. Also gated the connector across UE 5.3–5.8: the `mcp-version-matrix` workflow now runs these unit tests under each engine's bundled Python (3.9 on 5.3, 3.11 on 5.4+) as a pre-COMPILE `pyTestsPassed` gate, proving stdlib-only + syntax compat per version. Not yet gate-verified live (readiness/exit wait paths exercised end-to-end pending); scope-refined to gate on `ping` (the C++ `system.health` from `F-editor-readiness-probe` is not built yet).
