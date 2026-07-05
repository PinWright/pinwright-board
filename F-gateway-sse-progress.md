---
id: F-gateway-sse-progress
title: "Gateway: opt-in SSE responses with job progress notifications on POST /mcp"
status: OPEN
severity: Medium
category: feature
tags: [mcp, transport, sse, progress, jobs, parity-ue58]
---

# Gateway: opt-in SSE responses with job progress notifications on POST /mcp

The C++ gateway (`Transport\McpTransport.cpp`) is strictly request-response: it ignores `Accept`, always returns `application/json`, and long-running work is poll-only via `system.job_status`. Client research (July 2026) confirmed SSE is a strict superset, not a second protocol: the Streamable HTTP spec (rev 2025-03-26/2025-06-18, "Sending Messages to the Server") REQUIRES every conformant client to accept both `application/json` and `text/event-stream` on one endpoint, so plain JSON remains the default and nobody is locked out. Claude Code, Codex, Cursor 1.0+, Gemini CLI (httpUrl), VS Code 1.101+, Windsurf all speak streamable HTTP; Claude Desktop stays on the bundled stdio proxy.

Primary payoff: Claude Code renders `notifications/progress` in its tool UI AND progress resets its 5-minute idle abort (v2.1.187+), so long editor jobs (UBT runs, dump_folder sweeps, test runs) stop dying at the client timeout. Codex/Cursor/Gemini currently drop the notifications (legal, inert).

Scope constraints (from research; keep all):
- Upgrade a response to SSE ONLY when the request carries `_meta.progressToken` AND `Accept` includes `text/event-stream`; otherwise return `application/json` exactly as today.
- Stay stateless: never issue `Mcp-Session-Id` (immunity to the -32000 stale-session failure class on editor restarts).
- `GET /mcp` returns 405. No JSON-RPC batch. Bundled stdio proxy unaffected via Accept gating.
- Wire job progress events (the `Ctx.StartJob` ticket stream / jobs.jsonl source) into `notifications/progress` for the streaming request; final result terminates the stream.
- Epic parity note: UE 5.8's built-in server is SSE-always with a heartbeat-only counter; ours should send REAL job progress.

NOT in scope (user decision): sessions, `tools/list_changed`, MCP resources. (These are orthogonal to SSE: sessions are an optional spec feature, list_changed needs a standing GET stream and is useless for a single never-changing `call` tool, resources are plain request/response.)

Engine constraint (verified 2026-07-05): incremental response writes (`EHttpServerResponseFlags::MultipleWriteStream`/`HasAdditionalWrites`/`SkipHeaderWrite`, Runtime\Online\HTTPServer) are NEW in UE 5.8 and absent in 5.6/5.7 engine source. **Version-gate the SSE path to 5.8+**; 5.3-5.7 builds keep plain JSON + job polling unchanged. Do not build a custom socket listener for older engines. Epic's ModelContextProtocolServer.cpp is the working reference, including the synchronous write-callback reentry contract (HttpResultCallback.h) and mid-stream disconnect teardown (HttpConnection.cpp:183).

Job-system interplay (design decisions):
- Job tickets remain the canonical handle; SSE is a live view. First progress event on a stream must carry the job ticket id so a dropped stream degrades to polling.
- Do NOT infer block-and-stream from progressToken alone: Codex's rmcp auto-injects a progressToken on every call, which would flip all job RPCs to blocking for that client. Streaming a job to completion requires an explicit per-call opt-in (e.g. `wait: true`) in addition to token + Accept.

Acceptance: a long `system.run_tests` call from Claude Code with a progressToken streams progress and survives past 5 minutes; the same call from a plain-JSON client (no Accept: text/event-stream) behaves byte-identically to today.

## History
- `#1-sse-strict-superset` `OPEN` reporter — Gateway is JSON-only + poll-only; research confirms SSE via Accept+progressToken gating is a spec-mandated superset (no client lock-out) and Claude Code both renders progress and resets its 5-min idle abort on it. Add opt-in SSE with real job progress; stay stateless.
- `#2-version-gate-and-job-interplay` `OPEN` reporter — Verified engine streaming flags (MultipleWriteStream) are new in UE 5.8 and absent in 5.6/5.7: version-gate SSE to 5.8+, older engines unchanged. Added design constraints: job tickets stay canonical (stream embeds ticket id), and streaming a job to completion needs explicit per-call opt-in because Codex auto-injects progressToken on every call.
