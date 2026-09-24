---
id: B-stdio-proxy-wedged-behind-endless-stream
title: "stdio proxy serves one request at a time, so one never-ending streamed job hangs every later call silently"
status: OPEN
severity: High
category: bug
tags: [mcp, proxy, transport, sse, jobs, hang, timeout]
encounters: 1
costly: 1
lastSeen: 2026-09-24T01:10:00Z
---

# stdio proxy serves one request at a time, so one never-ending streamed job hangs every later call silently

`mcp_proxy.py` `serve_stdio` (line 2925) handles one JSON-RPC request at a time on one thread. A
streamed `tools/call` (block-and-stream: progress token + `Accept: text/event-stream`) is relayed
until its terminal frame; the SSE per-read timeout never fires because the server heartbeats and
emits a progress event every 10 s. If the job never becomes terminal (see
`B-run-tests-filter-job-never-terminal`), the proxy never returns, and every subsequent `call` on
that MCP connection queues behind it with no response, no progress and no error.

Observed (UE 5.8, host `unreal-fpv-new`, plugin `8748c637`): `editor.pie_status {}` and
`editor.list_dirty_packages {}` through `mcp__pinwright__call` each hung until Claude Code's
1800 s idle abort (`sent no response or progress for 1800s`). At the same moment the gateway
answered the same RPCs directly on `POST http://127.0.0.1:25565/mcp` in milliseconds, and
`system.job_list` showed one ticket, `system.run_tests j_20260923T210830_4e5912df`, `running` for
~4 h after its queue drained. The port was read from `Saved/PinWright/gateway-port` (25565) and
confirmed with `Get-NetTCPConnection -OwningProcess <editor pid> -State Listen`; note the
per-project derived port documented for this host (24281) was not the one bound. Proxy process:
PID 42424, started from `.mcp.json`, shared by the parent session and its subagents, so one stuck
stream froze the whole session tree. That the proxy was blocked on this particular stream is
inferred from the single-threaded loop plus the single running ticket, not observed in a proxy log.

Distinct from `F-gateway-sse-progress` `#10` (a bare screenshot returning `running` through the
proxy: a reporting gap, no hang).

**Workaround:** pass `wait: false` on job verbs so they return a ticket; if the connection is
already wedged, reconnect the MCP client (`/mcp`), which respawns the proxy.
**Fix (proposed):** give streamed relays an overall ceiling (or end the stream when the job has
emitted only elapsed-time progress past a limit and return the ticket), and/or serve requests
concurrently so one stream cannot block unrelated calls; at minimum log the in-flight method so a
wedge is diagnosable.

## History
- `#1-proxy-wedged-behind-stuck-job` `OPEN` reporter — Two calls hung 30 min each (costly: a
  subagent's 30 min and, per the orchestrator, a previous agent's ~3 h stall on the same session)
  while the gateway answered directly. Diagnosed and briefly worked around by calling the gateway
  over raw HTTP with per-call timeouts; that workaround was dropped because the plugin's
  CLAUDE.md forbids raw gateway calls, and the session went back to `mcp__pinwright__call`.
