---
id: B-mcp-claude-channel-capability-undeclared
title: "The server's initialize declares only tools.listChanged, and there is no claude/channel handler anywhere in the plugin — a client attempting that method finds nothing to answer it and nothing in the handshake said so"
status: OPEN
severity: Low
category: bug
tags: [mcp, transport, initialize, capabilities, protocol-conformance, discoverability, evidence-gap]
---

# `claude/channel` is neither declared nor implemented

`initialize` advertises exactly one capability. `Transport/McpRequestCore.cpp:282-300`:

```cpp
TSharedPtr<FJsonObject> BuildInitializeResult()
{
    TSharedPtr<FJsonObject> ToolsCap = MakeShared<FJsonObject>();
    ToolsCap->SetBoolField(TEXT("listChanged"), false);

    TSharedPtr<FJsonObject> Capabilities = MakeShared<FJsonObject>();
    Capabilities->SetObjectField(TEXT("tools"), ToolsCap);
    ...
    Result->SetStringField(TEXT("protocolVersion"), TEXT("2025-06-18"));
    Result->SetObjectField(TEXT("capabilities"), Capabilities);
```

`capabilities` contains exactly one key, `tools`, containing exactly one key, `listChanged: false`.
No `experimental`, no vendor namespace, no `claude/*`. The bundled stdio proxy's cold-start
`initialize` matches byte-for-byte in shape (`Content/Python/mcp_proxy.py:2292-2300`).

And there is no implementation to declare: **`claude/channel` has zero matches across the entire
repository**. A case-insensitive sweep for `claude` over `Plugins/PinWright/Source/` returns only
`CLAUDE.md` file references in comments (e.g. `Handlers/Actor/DynamicMeshCountProbe.h:24`,
`Handlers/AI/AIHandler.cpp:1762`) and two user-facing strings mentioning "Claude Code agents"
(`Handlers/Asset/AssetDumpHandler.cpp:2909`, `:3070`). No protocol method, no capability key, no
handler.

The five supported protocol methods are fixed and handled inside the transport rather than the
dispatcher: `initialize`, `notifications/initialized`, `ping`, `tools/list`, `tools/call`. Anything
else returns `-32601 Method not found`.

## Evidence gap, stated rather than glossed

The client-side observation that prompted this — reported as roughly 20 records across 9 of 9
sessions — **cannot be re-derived from this tree**. The claim came from a scan of MCP proxy logs
that is not on disk anywhere: `Docs/`, `Docs/map/`, `Docs/plugin-engineering/`,
`Plugins/PinWright/Docs/logs/` (which holds one unrelated 816 B file), `Plugins/PinWright/scratchpad/`
and the board were all searched, and no such record exists. The stdio proxy is no help either — it
logs relay failures only (`mcp_proxy.py:311-313`, *"Diagnostics go to stderr ONLY"*).

So this ticket asserts only the half that is re-derivable from source: **the server declares no
capability under which `claude/channel` could be answered, and implements no handler for it.**
Whoever picks it up should first re-establish the client-side frequency from a live client log
rather than trusting the 20/9-of-9 figure, which has the same unfalsifiable shape as the fabricated
count recorded in `B-blocked-on-modal-no-disk-trace`.

## Why it is Low

Nothing is broken for any caller today: the tool surface is a single `call` tool, no documented
workflow uses a `claude/*` method, and an unknown method gets a clean `-32601` rather than a hang or
a wrong answer. The cost is diagnostic noise and a handshake that does not describe the server's
actual surface.

## What a fix would decide

Either declare the vendor capability (MCP's `capabilities.experimental` is the conformant slot for
a non-standard namespace) if the channel is something the server should speak, **or** record
explicitly that it is a client-side probe the server is entitled to refuse — so the next forensic
pass recognises the `-32601` as expected rather than re-investigating it.

## Coverage note

`Tests/Infra/TestMcpRequestCore.cpp:188-202` pins the handshake: `capabilities` present,
`capabilities.tools` present, `listChanged` false. It asserts nothing about **other** capability
keys, so neither an undeclared vendor capability nor an accidentally-added one would be caught. A
fix should tighten that test to assert the full key set rather than a subset.

## History
- `#1-undeclared-and-unimplemented` `OPEN` reporter — `initialize` declares exactly one capability: `Transport/McpRequestCore.cpp:282-300` builds `capabilities` with the single key `tools`, itself carrying the single key `listChanged: false` — no `experimental`, no vendor namespace, no `claude/*`; the bundled stdio proxy's cold-start handshake matches byte-for-byte (`Content/Python/mcp_proxy.py:2292-2300`). There is also nothing to declare: **`claude/channel` has zero matches across the entire repository**, and a case-insensitive `claude` sweep over `Plugins/PinWright/Source/` returns only `CLAUDE.md` comment references and two user-facing strings in `AssetDumpHandler.cpp:2909,:3070`. The transport handles five fixed protocol methods (`initialize`, `notifications/initialized`, `ping`, `tools/list`, `tools/call`); anything else gets `-32601`. Rated Low because no caller is broken — the tool surface is a single `call` tool, no documented workflow uses a `claude/*` method, and an unknown method fails cleanly rather than hanging or answering wrongly. The cost is diagnostic noise plus a handshake that does not describe the server's real surface.
- `#2-the-client-side-count-is-not-re-derivable` `OPEN` reporter — Recording the evidence gap rather than letting the number stand unqualified. The observation that prompted this — roughly **20 records across 9 of 9 sessions** — comes from a scan of MCP proxy logs that **does not exist on disk**: `Docs/`, `Docs/map/`, `Docs/plugin-engineering/`, `Plugins/PinWright/Docs/logs/` (one unrelated 816 B file), `Plugins/PinWright/scratchpad/` and this board were all searched and no such record was found, and the stdio proxy logs relay failures only (`mcp_proxy.py:311-313`). This ticket therefore asserts only the source-derivable half: no capability is declared under which `claude/channel` could be answered, and no handler implements it. Re-establish the client-side frequency from a live client log before acting — the 20/9-of-9 figure currently has the same unfalsifiable shape as the fabricated "779 occurrences" recorded in `B-blocked-on-modal-no-disk-trace`, and that one turned out to be zero. Fix decision when someone picks this up: either declare it under `capabilities.experimental`, the conformant slot for a non-standard namespace, or record that it is a client-side probe the server is entitled to refuse so the next forensic pass reads the `-32601` as expected. Either way tighten `Tests/Infra/TestMcpRequestCore.cpp:188-202`, which asserts a SUBSET of the handshake keys and would catch neither an undeclared vendor capability nor an accidentally-added one.
