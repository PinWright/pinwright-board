---
id: F-mcpclient-toolset-compat
title: "Verify and document PinWright as a server for Epic's MCPClientToolset / EDA"
status: WONTFIX
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
- `#2-no-defect-needs-live-ue58-bench` `WONTFIX` developer — Not a fix-workflow ticket. Verified every transport premise already holds in HEAD, so there is NO code defect to fix: bearer auth matches Epic's `Authorization: Bearer <ApiKey>` (McpRequestCore.cpp:224-252 gate; MCPToolsetSettings.h:45-59 EMCPAuth::BearerToken default), `application/json` body returned (SocketHttpServer.cpp:749/760), single `call` tool via tools/list (McpRequestCore.cpp:310-317), protocolVersion "2025-06-18" (McpRequestCore.cpp:180). The core deliverable is a LIVE two-process UE-5.8 ↔ Epic-MCPClientToolset interop verification (initialize-negotiation acceptance of the hardcoded newer rev, same-process game-thread deadlock, single-call usability) — inherently human/interactive, not expressible as an in-process UE-5.7 automation test (the toolset type is 5.8-only). The docs + optional Setup-screen EDA entry are gated on that verification passing and are themselves UE-5.8-conditional (UMCPToolsetSettings is a 5.8 EditorPerProjectUserSettings UDeveloperSettings; unbuildable/unverifiable on this UE-5.7 host), so no code-actionable, in-process-verifiable subset exists. NOT a defer: UE 5.8 + MCPClientToolset are ALREADY installed here (C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\MCPClientToolset), so there is no engine-availability gate that lifts — the barrier is the fix-workflow's permanent no-live-editor / in-process-test-only design plus the absence of a code defect, and there is no nameable blocking ticket or justifiable review date (per board README, an ungateable defer is a WONTFIX). The FEATURE stays valid: a human should run the live UE-5.8 EDA interop bench and, if it passes, re-file a scoped code ticket (e.g. an EDA one-click entry writing the MCPToolsetSettings INI + a setup-docs section). The hardcoded protocolVersion is intentionally left alone — the ticket does not prescribe echo-negotiation, and changing it blind risks regressing today's working direct-connect clients (Claude Code/Cursor/etc.).
