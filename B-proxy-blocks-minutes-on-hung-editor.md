---
id: B-proxy-blocks-minutes-on-hung-editor
title: "mcp_proxy.py blocks up to 10 minutes per call when the editor is hung"
status: IN-REVIEW
severity: Medium
category: bug
tags: [mcp-proxy, transport, timeout]
---

# mcp_proxy.py blocks up to 10 minutes per call when the editor is hung

## Problem

The stdio<->HTTP bridge (`Content/Python/mcp_proxy.py`) forwards tool calls
with a 600s socket timeout (`--call-timeout` default). When the editor process
is alive but its game thread is blocked (hang, modal dialog, long load,
debugger break), the OS still accepts the TCP connection - UE's `IHttpRouter`
is ticked on the game thread and never answers - so the proxy blocks in
`urlopen` for the full 10 minutes. The proxy is single-threaded, so every
subsequent request (including client retries) queues behind the stuck one,
stalling the whole agent session for multiple back-to-back 10-minute waits.

The server-side request timeout (`HttpDefaultTimeoutMs` = 120s, swept in
`McpTransport.cpp`) cannot help: the sweep runs on the same hung game thread.
The docstring's graceful-degradation promise ("only individual tools/calls
degrade while the editor is down") holds for a dead editor (instant
connection-refused) but not for a hung one.

Repro: point a fresh proxy at a socket that accepts connections but never
responds, send one `tools/call` - the response takes the full call timeout.

## Expected

A dead or hung editor should fail each call within seconds with the graceful
in-band "editor not reachable" error, as the dead-editor path already does.

## History

- `#1-filed` (Reporter) Filed from a live session where pinwright calls hung
  for minutes against a non-responsive editor. Root cause traced to the 600s
  forward timeout plus the game-thread-ticked HTTP router never answering.
- `#2-fixed-probe-and-timeout` (Developer) Fixed in `Content/Python/mcp_proxy.py`:
  added a pre-flight JSON-RPC `ping` liveness probe (new `--probe-timeout`,
  default 5s) before every forwarded request - probe failure returns the
  graceful in-band error immediately, with distinct wording for
  connection-refused (not running) vs probe timeout (hung/busy/starting);
  lowered `--call-timeout` default 600s -> 150s (the editor's own request sweep
  fires at 120s, so a healthy editor always answers within that). Verified with
  a spawned-proxy harness: dead endpoint errors in ~2s (Winsock refused-connect
  floor), accept-but-never-respond endpoint errors in ~5s (previously 600s),
  and a healthy fake endpoint forwards normally with no measurable probe
  latency. Live-editor smoke leg still worth a pass by the tester (editor was
  not running during verification; probe/forward use the same unchanged
  `_post` + token wiring).
