---
id: F-gateway-bearer-token-auth
title: "Bearer-token auth for the MCP HTTP gateway (file handshake)"
status: IN-REVIEW
severity: High
category: feature
tags: [security, transport, onboarding]
---

# Bearer-token auth for the MCP HTTP gateway (file handshake)

The gateway (`POST /mcp`, loopback-only) has zero authentication. Any local
process, and critically any web page via preflight-free CSRF (`Content-Type:
text/plain` POST) or DNS rebinding, can POST to `http://127.0.0.1:<port>/mcp`
and drive ~1,200 RPCs that amount to arbitrary code execution
(`editor.console_command`, `editor.launch_standalone`, `pipeline.run_ubt`,
arbitrary file writes). The derived port range (19880-30119) is sprayable from
browser JS in seconds, so port obscurity is no defense.

Threat model: block browsers (cannot read local files) and other OS users.
Same-OS-user processes are inside the trust boundary by design.

**Fix:** Require `Authorization: Bearer <token>` on every request.

- Token: 32 CSPRNG bytes (OpenSSL `RAND_bytes`), lowercase hex, persisted at
  `Saved/PinWright/gateway-token` (atomic write, POSIX 0600, regenerated only
  if missing/empty). Rotation = delete file + restart editor.
- Gate in the `McpTransport` route lambda before body/parse; 401 +
  `WWW-Authenticate: Bearer` + self-diagnosing JSON-RPC error naming the token
  path and the setup screen.
- `bRequireAuthToken` setting (default ON). When OFF: Install/Update write
  configs WITHOUT token plumbing, server ignores the Authorization header
  entirely (old tokened configs keep working); when ON, tokenless configs get
  401 and Detect reports them Outdated.
- Onboarding writers bake `--token-file <path>` into stdio-proxy args and
  `headers.Authorization` into direct-HTTP configs; Detect treats missing
  token plumbing as Outdated so the Install button is the migration path.
- `mcp_proxy.py` reads the token file per request (`--token-file` arg plus a
  script-relative fallback so existing stdio installs migrate with no
  re-install). Security invariant: the proxy never reads a token path supplied
  by the server (port-squatter file-exfiltration hazard).

Out of scope: Origin/Host/Content-Type validation (rejected alternative),
rotation UI, rate limiting, configurator unit tests, Windows ACL hardening of
the token file.

## History
- `#1-filed-bearer-token-design` `OPEN` reporter — gateway has no auth; browser CSRF/DNS-rebinding drive-by = RCE; design settled (bearer token + file handshake, kill switch, onboarding migration), implementation plan approved
- `#2-implemented-bearer-gate` `IN-REVIEW` developer — "Added Utils/GatewayAuthToken (OpenSSL RAND_bytes → 64-hex, Saved/PinWright/gateway-token, atomic tmp+Move, POSIX 0600, OR-accumulate constant-time compare, test root-override seam). 401 bearer gate in McpTransport route lambda before parse (reuses JsonRpc::BuildErrorResponse/kInvalidRequest + McpHttpUtils::BuildJsonHttpResponse; WWW-Authenticate: Bearer; lowercase 'authorization' lookup since UE lowercases header keys). Start() gained defaulted InAuthToken param (test callers stay auth-off). bRequireAuthToken setting (default ON). Subsystem resolves token gated on setting, fails open with Error log. AgentMcpConfigurator: FServerLaunch TokenFilePath/TokenValue, gated resolve, --token-file in stdio args, headers.Authorization in Claude/Gemini/VSCode/Cursor http variants (Codex http left TODO — no confirmed TOML header key; stdio carries it), Detect tightened to flag tokenless configs Outdated only when auth on. mcp_proxy.py: --token-file + script-relative fallback (per-request read, security invariant no server-supplied path, 401 id-patch). Tests: PinWright.infra.gateway_auth_token.* (5) + PinWright.infra.mcp_transport.Auth.* (5, save/restore shared token). Docs updated; removed false 600 req/min rate-limit line. Not yet compiled/tested by user."
