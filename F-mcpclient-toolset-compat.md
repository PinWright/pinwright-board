---
id: F-mcpclient-toolset-compat
title: "Verify and document PinWright as a server for Epic's MCPClientToolset / EDA"
status: OPEN
severity: Medium
category: feature
tags: [mcp, interop, eda, epic, parity-ue58]
---

# Verify and document PinWright as a server for Epic's MCPClientToolset / EDA

UE 5.8 ships MCPClientToolset (Beta, `C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\MCPClientToolset\`), an outbound MCP client that imports external MCP servers' tools into the ToolsetRegistry, which feeds the Epic Developer Assistant and Epic's MCP re-export. If PinWright registers cleanly, EDA users can drive PinWright's surface without installing Claude Code/Cursor: a distribution checkbox ("works with Epic Developer Assistant") for the Fab listing.

Confirmed from 5.8.0 source (`MCPToolsetSettings.h`): auth enum { None, BearerToken (default), OAuth2 PKCE }; ApiKey is sent as `Authorization: Bearer <ApiKey>` (comment at :45), which matches PinWright's existing bearer-token gateway as-is. Transport options: Legacy SSE (default) | StreamableHTTP; under StreamableHTTP the MCP spec requires clients to accept plain `application/json` responses, which is what PinWright returns.

To verify (live editor test):
1. Configure Editor Preferences -> Plugins -> MCP Toolset Servers with PinWright's URL + token (StreamableHTTP mode); confirm the single `call` tool imports.
2. Protocol-rev tolerance: PinWright speaks 2025-06-18; Epic's client targets 2025-03-26+. Confirm initialize negotiation succeeds.
3. Same-process deadlock check: EDA executes tools on the game thread while PinWright's server also marshals to the game thread in the SAME editor process; verify Epic's HTTP tool call is async (no deadlock) before documenting.
4. Single-`call`-tool usability: can EDA's model learn the wiki drill-in pattern in-band?

Then: setup docs section + optional EDA entry in the PinWright Setup screen's one-click installs.

**Workaround:** none needed; external agents connect directly today.

## History
- `#1-eda-interop-untested` `OPEN` reporter — Epic's MCPClientToolset (Bearer auth default, StreamableHTTP option) should consume PinWright as-is, but protocol-rev negotiation, same-process game-thread deadlock, and single-call-tool usability by EDA are unverified. Verify, then document + optional setup-screen entry.
