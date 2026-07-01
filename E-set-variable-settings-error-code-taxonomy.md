---
id: E-set-variable-settings-error-code-taxonomy
title: "Make dispatcher request lifecycle observable in PDS.log + enrich TIMEOUT message"
status: DONE
severity: Low
category: ergonomic
tags: [dispatcher, transport, observability, logging]
---

# Make dispatcher request lifecycle observable + enrich TIMEOUT message

When a tool call appears to fail with Claude Code's client-side string
`[Tool result missing due to internal error]`, the operator has no
plugin-side trail to reconstruct what happened. That string is NOT
produced by this plugin (`grep` across plugin source returns zero hits;
it is emitted by the MCP client when a child-process response is missing
or malformed).

The dispatcher already logs handler completion at `Verbose`
(`RpcDispatcher.cpp:174`), but it does NOT log entry. The transport
emits `TEXT("TIMEOUT")` with no method context (`RpcTransport.cpp:492`),
making it impossible to tell from a `TIMEOUT` what RPC stuck.

## Scope of this ticket

1. **Handler entry log.** Emit a matching `Verbose` entry log in
   `FRpcDispatcher::ProcessRequest` immediately before invoking the
   handler. Format: `"Dispatching '%s' (id=%s)"`. Pairs with the
   existing exit log so `-log -verbose` runs show every request lifecycle.

2. **Enrich TIMEOUT message and remember method on completion.** Add an
   optional `Method` field to `FPendingTransportCompletion`, set it in
   `RegisterCompletion` from a new optional argument, and include it in
   the timeout message: `"Request timed out after %.0fs (method=%s)."`.
   Error code stays `TIMEOUT` for back-compat. Dispatcher passes the
   method through when it is the registrant; HTTP route registrations
   in the transport already know the method and pass it too.

## Out of scope

- SEH/__try wrapping of the handler invocation. Tracked separately as
  `E-dispatcher-seh-crash-handler` (to be filed). Requires a no-destructor
  shim, MSVC-only conditional compile, and a flaky null-deref test
  handler — not a sprint-sized change.
- Node MCP bridge changes. Audited the bridge: `mcp-server/src/index.js`
  does not batch JSON-RPC requests, so the proposed "synthesize errors
  for missing batch entries" fix would be no-op. The opaque
  `[Tool result missing]` string is produced by Claude Code itself when
  the MCP child-process stdout response to a `tools/call` is missing,
  which is outside this plugin's surface.
- Distinguishing TIMEOUT_CLIENT_DISCONNECT from TIMEOUT_HANDLER_STUCK.
  UE's `IHttpRouter` does not expose a disconnect signal observable
  from `RegisterCompletion`, so we cannot reliably split them. Punted.

## Acceptance criteria

- A request to a registered handler emits two `LogRpcDispatcher Verbose`
  lines: one before the handler runs (`Dispatching '<method>' (id=<id>)`)
  and the existing one after (`Handler '<method>' OK ... ms`).
- A timeout error message contains the duration and method name, e.g.
  `Request timed out after 120s (method=blueprint.set_variable_settings).`
- Error code remains `TIMEOUT` (callers depending on the exact code
  string keep working).

## History
- `#1-split-from-transient-bug` `OPEN` reporter — Split out from
  `B-blueprint-set-variable-settings-transient-internal-error` during
  mcp-sprint analysis. Original ticket closed as WONTFIX because the
  underlying plugin handler is provably total on responses, and the
  transient failure is almost certainly either in the Node MCP bridge or
  an uncatchable SEH in nested UE code. This ticket captures the actual
  plugin-side improvement: better failure taxonomy so opaque
  `[Tool result missing]` becomes a typed error code. See the original
  ticket for evidence dump (dispatcher/transport audit).
- `#2-reformulated-narrow-scope` `OPEN` developer — Reformulated after Phase 2 audit. Original sub-items #1 (SEH) and #4 (TIMEOUT split) punted as separate future tickets; sub-item #2 (Node bridge batch fix) rejected as based on a false premise (the bridge does not batch). Remaining scope: entry log + enriched TIMEOUT message.
- `#3-entry-log-and-method-aware-timeout` `IN-REVIEW` developer — Added Verbose entry log "Dispatching '<method>' (id=<id>)" in RpcDispatcher::ProcessRequest. Added Method field to FPendingTransportCompletion plumbed through RegisterCompletion; timeout message now includes "(method=<method>)". Error code stays TIMEOUT. Tests added in TestDispatcher.cpp and TestRpcTransport.cpp.
- `#4-skip-needs-log-or-timeout` `SKIP` tester — Both acceptance criteria require either inspecting `PDS.log -log -verbose` output or triggering a real handler timeout (>120 s wait). Neither is feasible to validate live in this MCP session: log inspection happens out-of-band, and inducing a timeout would block the editor. Implementation references in `TestDispatcher.cpp` / `TestRpcTransport.cpp` exist; trusting the unit tests to cover both criteria. Leaving IN-REVIEW for an out-of-band log spot-check.
- `#5-accepted-without-recheck` `DONE` tester — Accepted by user decision without further live verification; prior SKIP entry documents why log/timeout validation was not re-run.
