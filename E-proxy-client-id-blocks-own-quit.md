---
id: E-proxy-client-id-blocks-own-quit
title: "Each stdio proxy process gets a new X-PinWright-Client id, so a script client's own earlier calls make editor.quit / editor_restart refuse with EDITOR_IN_USE; editor_restart has no force"
status: OPEN
severity: Low
category: ergonomic
tags: [mcp-proxy, editor-quit, editor-restart, editor-in-use, client-id]
encounters: 3
lastSeen: 2026-09-29T11:20:00Z
---

# A script's own traffic blocks its own restart

A client that spawns `mcp_proxy.py` per invocation (a normal pattern for script-driven sessions) gets a fresh `_CLIENT_ID` each time. `editor_restart` then fails with `EDITOR_QUIT_REFUSED ... [EDITOR_IN_USE] ... served 'editor.list_dirty_packages' for a different client 15s ago`, where the "different client" was the same agent one process earlier. `editor_restart` has no `force`, so the only path is `editor.quit {force: true}` followed by `editor_start`.

**Fix:** let the client pin its id (env var or `--client-id`), and add `force` to `editor_restart`.

**Source (8748c637):** `_CLIENT_ID = uuid.uuid4().hex` is minted once per proxy process (`Content/Python/mcp_proxy.py:304`). `_request_editor_quit` (`:2199-2215`) deliberately omits `force`, and its docstring asserts "This proxy's own calls never trigger it", which is false when the same client spawns a proxy per invocation.

## History
- `#1-own-traffic-blocks-restart` `OPEN` reporter — UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`.
- `#2-anonymous-http-caller-variant` `OPEN` reporter - Variant, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv-wt1` (Linux), plugin `61c243f5`. Same session, same MCP proxy for `editor.quit`, but some earlier `property.get` calls went straight to `http://127.0.0.1:27673/mcp` with the bearer token and no `X-PinWright-Client` header (a bash `curl` helper for scripted readbacks). `editor.quit {}` then refused: `[EDITOR_IN_USE] ... served 'property.get' for a different client 130s ago` with `"lastClient":""`. An empty client id is treated as someone else, and the message does not say the other caller was anonymous. `force:true` worked. Cheap. Ask: say "an anonymous (no client id) caller" when `lastClient` is empty, so the caller can recognise its own traffic.
- `#3-anonymous-http-caller-again` `OPEN` reporter - Same anonymous-caller variant, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux). A python urllib helper called `object.call_function` directly over HTTP without `X-PinWright-Client`; 91 s later `editor.quit {discard:true}` from the MCP session refused with `EDITOR_IN_USE` and `"lastClient":""`. `force:true` worked. No lost work.
