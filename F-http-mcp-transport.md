---
id: F-http-mcp-transport
title: "Add shared HTTP MCP transport for editor-automation and agent-testing"
status: DONE
severity: High
category: feature
tags: [mcp, http, transport, codex, claude, agent-testing]
---

# Add shared HTTP MCP transport for editor-automation and agent-testing

Codex Desktop launches stdio MCP server processes per active session or agent.
With many agents, the `editor-automation` and `agent-testing` MCP wrappers can
fan out into many Node processes even though both wrappers only proxy to one
Unreal RPC backend. Add opt-in Streamable HTTP MCP transport to both wrappers so
Codex and Claude can share one long-lived local MCP service per wrapper instead
of spawning one stdio process per agent.

Scope:
- `Plugins/EditorAutomationRpcGateway/mcp-server/src/index.js`
- `Plugins/AgentTesting/mcp-server/src/index.js`
- Package scripts, docs, and config snippets needed to run the HTTP services
  from both Codex and Claude.

Required behavior:
- Keep stdio as the default and fully supported transport for existing Claude
  Desktop, CLI, and one-shot workflows.
- Add `--http --port <port>` mode for both wrappers using the MCP Streamable
  HTTP transport.
- Treat Codex and Claude Code compatibility as a hard requirement, not a
  best-effort target. Do not land an HTTP mode that only works in one client.
- Keep backend RPC URLs unchanged (`UE_RPC_URL` / `19880`,
  `AGENT_TESTING_URL` / `19881`); HTTP MCP wrapper ports must be separate local
  ports.
- Create a fresh MCP server/session per HTTP client session inside the
  long-lived process, or otherwise prove the SDK transport is safe for
  concurrent clients.
- Handle `SIGINT` / `SIGTERM` clean shutdown.
- Preserve tool schemas and result formats exactly.

Acceptance:
- Existing stdio startup still works.
- `editor-automation` can serve `/mcp` on a local port and return `tools/list`
  plus the `call` tool.
- `agent-testing` can serve `/mcp` on a local port and return `tools/list` plus
  screenshot/input tools.
- Codex config can use `url = "http://127.0.0.1:<port>/mcp"` for both wrappers
  with no per-agent duplicate wrapper process, including any required local
  bearer/header configuration.
- Claude Code can use the same local HTTP MCP URL with equivalent header
  configuration.
- Claude Desktop or legacy Claude workflows can still use the stdio transport if
  their HTTP/header support is unavailable or unreliable.
- Documentation includes example Codex config, Claude Code config, and stdio
  fallback config, plus a note that Unreal backend traffic still targets one
  editor instance and mutating calls may need queueing if many agents use it
  concurrently.

**Workaround:** Lower Codex agent parallelism or run fewer stdio MCPs. QMD can
already be served over HTTP separately.

## History
- `#1-stdio-process-fanout` `OPEN` reporter — Many concurrent agents cause stdio MCP wrappers to fan out into many Node processes. Track an opt-in HTTP transport for `editor-automation` and `agent-testing` so one local service per wrapper can be shared by Codex/Claude while retaining stdio compatibility.
- `#2-add-streamable-http-transport` `IN-REVIEW` developer — Added opt-in `--http --port` mode to both `mcp-server/src/index.js` wrappers using stateless `StreamableHTTPServerTransport` (per-request `McpServer`+transport pair, no session map). Stdio remains the default and is byte-equivalent. Auth via `--auth-token` or `MCP_HTTP_AUTH_TOKEN`. SIGINT/SIGTERM close the HTTP server with a 5s force-exit fallback. Added `start:http` package scripts (ports 19882/19883), per-wrapper acceptance tests under `test/test-http-transport.js`, and Codex/Claude Code HTTP config docs in `EditorAutomationRpcGateway/CLAUDE.md` and `AgentTesting/README.md`.
- `#3-verify-fix` `DONE` tester — Verified: ran `node --test test/test-http-transport.js` for both wrappers on ephemeral ports (`--port 0`). editor-automation: `initialize` returned `serverInfo.name=editor-automation` and `tools/list` exposed `call` (2 pass / 1 SIGTERM-skip on win32). agent-testing: same shape — `serverInfo.name=agent-testing`, `tools/list` exposed the six `agent.*` tools (2 pass / 1 SIGTERM-skip). Had to `npm install` in `Plugins/AgentTesting/mcp-server` first (deps were not vendored); editor-automation deps were already present.
