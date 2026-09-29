---
id: E-call-string-args-rejected
title: "call rejects `args` sent as a JSON-encoded string, which some MCP clients produce for nested objects, so every executing call from those clients fails"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [transport, mcp, call, args, client-compat, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# call rejects a JSON-string `args`

The single MCP tool `call(method, args)` declares `args` as `{"type": "object"}`
(`Source/PinWright/Private/Transport/McpRequestCore.cpp:249-255`, and the proxy's copy in
`Content/Python/mcp_proxy.py:156-161`). The `tools/call` handler enforces it strictly
(`McpRequestCore.cpp:724-738`, plugin HEAD `71c91649`):

```cpp
if (!ArgsValue.IsValid() || ArgsValue->Type != EJson::Object)
{
    ... JsonRpc::kInvalidParams, "tools/call argument 'args' must be an object when present."
```

Some MCP clients serialize a nested object argument as a JSON string, so the editor receives
`"args": "{\"assetPath\":\"/Game/...\"}"`. For those clients every call that executes an RPC fails
with `-32602`, while wiki lookups (no `args`) still work, which makes the integration look half
broken. Another Unreal MCP server handles this case explicitly: when its nested params field is a
string it deserializes it and accepts the result if it is an object, with a comment that Claude Code
serializes objects to strings.

The stdio proxy has the same blind spot one layer up: its streaming decision reads
`args.get("wait")` only when `args` is a dict (`mcp_proxy.py:392-394`), so a string `args` silently
takes the default path.

Engines: not engine-dependent.

**Fix:** in `McpRequestCore.cpp`, when `args` is `EJson::String`, run it through
`FJsonSerializer::Deserialize`; if that yields an object, use it, otherwise keep the current
`-32602` with a message that says the string did not parse to an object. Apply the same coercion in
`mcp_proxy.py` before it inspects `args`. Leave `arguments` (the outer MCP field) strict. Add
transport tests for an object-in-string, a non-object JSON string, and invalid JSON.

**Related:** `E-call-unknown-arg-fields-silently-ignored`, `B-param-type-never-validated`.

## History
- `#1-string-args-rejected` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis and re-verified at plugin HEAD `71c91649`: `McpRequestCore.cpp:731-735` rejects any non-object `args`, including a string holding a valid object. Not reproduced with a stringifying client. Filed as ergonomic: the input violates the declared schema, but the fix is cheap and removes a hard blocker for affected clients. Severity Medium: every executing call fails for those clients (hard blocker class), bumped down because it affects only some clients.
- `#2-string-args-coerced` `IN-REVIEW` developer - "`McpRequestCore.cpp` `tools/call`: a string `args` is run through `FJsonSerializer::Deserialize` and dispatched as the object when it parses to one (so `args.wait` is honoured); a non-object JSON string or invalid JSON stays `-32602` with 'did not parse to a JSON object'; outer `arguments` stays strict. `mcp_proxy.py` gains `_normalize_string_args`, applied to every forwarded `tools/call` before `_wants_stream`, leaving non-object strings for the editor to reject. Tests: C++ `PinWright.infra.request_core.ToolsCall.StringArgs` (object string incl. wait=false, array/number/invalid-JSON strings rejected, outer object-string still rejected), Python `test_mcp_proxy_sse.NormalizeStringArgsTest` (4 tests). Wiki: `docs/wiki-src/mcp-transport.md` argument-shapes section. Verified: bundled-Python `unittest discover tests` 264 OK (1 skip); `-SingleFile` compile of both C++ files on 5.8 succeeded. Not verified: the C++ test run and a live call from a stringifying client."
