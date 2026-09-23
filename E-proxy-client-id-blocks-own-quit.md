---
id: E-proxy-client-id-blocks-own-quit
title: "Each stdio proxy process gets a new X-PinWright-Client id, so a script client's own earlier calls make editor.quit / editor_restart refuse with EDITOR_IN_USE; editor_restart has no force"
status: OPEN
severity: Low
category: ergonomic
tags: [mcp-proxy, editor-quit, editor-restart, editor-in-use, client-id]
encounters: 1
lastSeen: 2026-09-23T18:30:00Z
---

# A script's own traffic blocks its own restart

A client that spawns `mcp_proxy.py` per invocation (a normal pattern for script-driven sessions) gets a fresh `_CLIENT_ID` each time. `editor_restart` then fails with `EDITOR_QUIT_REFUSED ... [EDITOR_IN_USE] ... served 'editor.list_dirty_packages' for a different client 15s ago`, where the "different client" was the same agent one process earlier. `editor_restart` has no `force`, so the only path is `editor.quit {force: true}` followed by `editor_start`.

**Fix:** let the client pin its id (env var or `--client-id`), and add `force` to `editor_restart`.

**Source (8748c637):** `_CLIENT_ID = uuid.uuid4().hex` is minted once per proxy process (`Content/Python/mcp_proxy.py:304`). `_request_editor_quit` (`:2199-2215`) deliberately omits `force`, and its docstring asserts "This proxy's own calls never trigger it", which is false when the same client spawns a proxy per invocation.

## History
- `#1-own-traffic-blocks-restart` `OPEN` reporter — UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`.
