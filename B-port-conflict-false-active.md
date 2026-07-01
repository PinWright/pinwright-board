---
id: B-port-conflict-false-active
title: "Transport reports active even when port bind fails (port in use)"
status: IN-REVIEW
severity: High
category: bug
tags: [transport, http, multi-editor]
---
# Transport reports active even when port bind fails (port in use)

When a second editor starts with the same MCP port (default 19880) — common in the
multi-checkout workflow — the in-process HTTP listener fails to bind, but the plugin
reports the server as active anyway.

Root cause: `FMcpTransport::Start()` calls `GetHttpRouter(port)` before listeners are
enabled, so the engine returns a valid router without binding (the `!HttpRouter.IsValid()`
guard never catches a conflict). The real `socket->Bind()` happens inside
`FHttpServerModule::StartAllListeners()`, which **ignores** the failed `StartListening()`
return value. `Start()` then unconditionally sets `bHttpServerActive = true` and logs a
false "MCP server active". Result: the second editor's `IsActive()` returns true,
`GetPort()` returns the configured port, and it writes that port to its own `port.txt` —
all lying. Every agent request to the port is silently served by the **first** editor, so
an agent that believes it is automating editor B is actually driving editor A, with no
error anywhere.

**Fix:** A pre-flight loopback bind probe in `Start()` before `GetHttpRouter()`. If the
port is already taken, set a new `bPortConflict` flag, leave `bHttpServerActive` false,
and log an error instead of claiming success. The subsystem exposes the resolved state via
`GetServerStatus()` → `EMcpServerStatus { Disabled, Listening, PortInUse }`, and the setup
screen shows a status banner (green listening / red conflict / amber disabled). On a
conflict the screen auto-opens at startup, gated by a new
`bShowSetupScreenOnConnectionFailure` setting (default true).

## History
- `#1-initial-repro` `OPEN` reporter — Two editors on the same port: the second silently fails to bind but reports active; agents misroute to the first editor.
- `#2-preflight-probe-and-status` `IN-REVIEW` developer — Added a pre-flight loopback bind probe + `bPortConflict`/`DidPortConflict()` in `McpTransport.cpp/.h`; `GetServerStatus()` + `EMcpServerStatus` on the subsystem; status banner and second show-condition checkbox on `SGatewaySetupScreen`; conflict-gated auto-open in `EditorAutomationRpcGatewayModule.cpp`; `Sockets` dep in `Build.cs`.
- `#3-per-project-port-now-opt-in` `IN-REVIEW` developer — The port defaults to a fixed `19880` (same on every machine, so committed agent configs stay portable), which means the original same-port collision scenario is the default behavior — handled by this ticket's pre-flight probe / refuse-to-serve. **Auto-derive Port From Project Path** (`bAutoDerivePort`, default off) is the opt-in remedy: enabling it derives `19880 + hash(projectPath) % 10240` (`19880`–`30119`), giving checkouts at different paths distinct ports. The runtime `port.txt` discovery file was removed (it had no readers); onboarding bakes the resolved endpoint URL into each agent's project-local config. On a rare cross-folder hash collision (or an OS-reserved port) even with auto-derive on, the user sets a different fixed `HttpPort` and re-runs onboarding.
- `#4-auto-derive-now-default-on` `IN-REVIEW` developer — Reversed `#3`'s default: `bAutoDerivePort` now defaults **on** (flipped in the `UPinWrightSettings` ctor), so each project folder gets a distinct path-derived port (`19880`–`30119`) out of the box and the fixed-`19880` same-port collision from `#1` is no longer the default path. This ticket's pre-flight probe still guards the remaining cases (auto-derive disabled for team-portable configs, or a rare cross-folder hash collision). Docs updated to match: README, plugin CLAUDE.md, `docs/wiki-src/mcp-transport.md`.
