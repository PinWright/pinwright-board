---
id: B-mcp-progress-token-never-echoed
title: "Every notifications/progress frame this server ever emitted was discarded by the client — the transport parsed the caller's params._meta.progressToken and threw it away, then sent the server's own job ticket id in its place; 83 rejections over five days, 100% failure, each also tearing down and silently reconnecting stdio"
status: IN-REVIEW
severity: High
category: bug
tags: [mcp, transport, sse, progress, protocol-conformance, silent-failure, stdio, jobs, spec-2025-06-18]
---

# The progress token was invented, so no client could ever correlate a frame

MCP `2025-06-18` (Progress, "Behavior Requirements") admits progress notifications only for tokens
**"provided in an active request"**. The server sent its own job ticket id as `progressToken`, which
is a token the client had never issued — so a conforming receiver has no request to attribute the
frame to and is entitled to discard it. Every frame this server had ever emitted was discarded.

The parse was never missing. `Transport/McpRequestCore.cpp:802-809` read
`params._meta.progressToken` into `FRequestDecision::ProgressToken` and has done so unchanged since
before the fix. The value was then dropped one layer up: `FPendingCompletion` had nowhere to put
it, `SocketHttpServer.cpp:923` did not forward it, and the only consumer of the parsed token was the
boolean gate `return Decision.bHasProgressToken;` (`ShouldStreamRequest`, `SocketHttpServer.cpp:950-969`).

The whole defect on the emit side was one line in `HandleJobEvent`:

```cpp
    ParamsObj->SetStringField(TEXT("progressToken"), TicketId);
```

## Measured

From this checkout's own MCP client log: **83 rejections over five days, 2026-08-12 to 2026-08-16**,
every one reading `Received a progress notification for an unknown token`, and **each of them
additionally tore down and reconnected the stdio transport**. A 100% failure rate on a feature that
was fully wired, invisible because both ends failed quietly — the client dropped the frame, the
server logged nothing.

The bundled proxy is not implicated and did not mask it: `Content/Python/mcp_proxy.py:2170-2192`
relays `notifications/progress` to the client **verbatim**, so the invented token went straight
through to where the rejection happened.

## Why nothing caught it

The pre-existing SSE tests **inject the notification they later assert on**
(`Tests/Transport/TestSseFraming.cpp:65,109-110` and `Tests/Transport/TestStreamLifecycle.cpp:50,58,94-95`,
both building the frame themselves via `SocketTestClient.h:558-563 BuildProgressNotification`). They
round-trip the **framing** and never observe which token the server would have chosen. The remaining
coverage (`Tests/Transport/TestStreamGate.cpp`) is gate-only — it asserts that a token's *presence*
upgrades the response, which is precisely the one thing that did work.

## Consequence for `F-gateway-sse-progress`

That feature ticket sits at `IN-REVIEW` with eight history entries, `#8` claiming live end-to-end
verification through Claude Code -> stdio proxy -> editor. The rejections were already accumulating
when `#8` was written. Nothing in its eight entries mentions the token substitution, because the
observable it checked — the call blocks and returns its terminal result — is downstream of the
progress frames and unaffected by their being dropped. **The feature was 100% ineffective at the
one thing it is named for**, and its verification could not see that.

## Fix (shipped `30b86d26`)

- The token is retained verbatim on `FPendingCompletion` (`SocketHttpServer.h:136-141`, string or
  number — MCP permits both), forwarded at `SocketHttpServer.cpp:923-924`, stored at `:1007`, and
  handed back by the new `FSocketHttpServer::GetProgressToken` (`:1081-1092`, declared `.h:74-76`).
- `PinWrightSubsystem.cpp:435-454` echoes the client's token and keeps the ticket id in its own
  `ticket_id` field, so a dropped stream still degrades to polling `system.job_status`.
- **Two further conformance defects in the same 25 lines.** No verb set a progress number, so every
  frame fell back to the ticket's progress **array length** — ring-trimmed to 50 — and a long job
  counted to 50 and then reported 50 forever, against a spec MUST; the wire value now derives from
  `FJobTicket::ProgressSeq`, which is never trimmed. And `total` was never sent, so no client could
  draw a bar; it is now published when the reporter knows its denominator and **omitted, not
  zero-filled**, when it does not.
- Where the spec and honesty collide, honesty wins, documented at `rpc-design.md` §9a: MCP says
  progress MUST increase, but `asset.dump_folder` legitimately sits flat in "waiting for async
  compilation", so incrementing to satisfy the letter would draw a bar advancing while nothing
  happened. A reporter that goes backwards is **held** at its previous value, never advanced past
  it, and the hold is disclosed as `reportedProgress` + `progressHeldAtPrevious`.

## Verification

`Tests/Transport/TestProgressTokenEcho.cpp`:
`PinWright.transport.progress_token.StreamRetainsTheClientsToken` (`:20-71`) asserts the retained
token is **byte-identical** to the one the client sent (`:63-64`) with a failure-direction check at
`:67-69`; `PlainRequestRetainsNoToken` (`:76-125`) asserts no token is invented for a plain request
(`:122-123`). Live: 400 assets loaded from disk in 23.64 s, 20 frames carrying progress 0..380 of
total 400, **zero client rejections after the fix against 83 before**.

## The transferable rule

`rpc-design.md:195` — **parsing a protocol field and discarding it is worse than not parsing it.**
The gate passes, so everything downstream looks wired.

## Related

- `F-gateway-sse-progress` — the feature this made ineffective.
- `B-job-progress-interval-default-60s` — same symptom (progress not arriving), unrelated cause,
  filed separately on purpose: that one degrades the polling path, which has no MCP involvement.

## History
- `#1-every-frame-discarded` `OPEN` reporter — Every `notifications/progress` frame the server had ever emitted was being discarded by the client. `HandleJobEvent` sent the server's own job ticket id as `progressToken` (`ParamsObj->SetStringField(TEXT("progressToken"), TicketId);`), but MCP 2025-06-18 admits only tokens "provided in an active request", so the receiver had no request to attribute the frame to. The transport DID parse the caller's token — `McpRequestCore.cpp:802-809` reads `params._meta.progressToken` into `FRequestDecision::ProgressToken`, unchanged for the whole life of the defect — and then threw it away, because `FPendingCompletion` had nowhere to put it and `SocketHttpServer.cpp:923` did not forward it; its only consumer was the boolean gate `return Decision.bHasProgressToken;`. Measured in this checkout's own MCP client log: **83 rejections over five days, 2026-08-12 to 2026-08-16**, every one reading `Received a progress notification for an unknown token`, and **each additionally tearing down and reconnecting the stdio transport**. A 100% failure rate on a fully wired feature, invisible because both ends failed quietly. The bundled proxy is not implicated — `Content/Python/mcp_proxy.py:2170-2192` relays the frame verbatim.
- `#2-why-the-suite-could-not-see-it` `OPEN` reporter — Recorded because the coverage gap is the reusable part. The pre-existing SSE tests **inject the notification they later assert on** (`Tests/Transport/TestSseFraming.cpp:65,109-110`, `Tests/Transport/TestStreamLifecycle.cpp:50,58,94-95`, both via `SocketTestClient.h:558-563 BuildProgressNotification`), so they round-trip the framing and never observe which token the SERVER would choose; the rest (`TestStreamGate.cpp`) asserts only that a token's presence upgrades the response, which is the one part that worked. Consequence for the parent feature: `F-gateway-sse-progress` sits `IN-REVIEW` with `#8` claiming live end-to-end verification through Claude Code -> stdio proxy -> editor, written while these rejections were already accumulating — its observable (the call blocks and returns a terminal result) is downstream of the progress frames and unaffected by their being dropped, so the feature was 100% ineffective at the thing it is named for and its own verification could not see that.
- `#3-fix` `IN-REVIEW` developer — Fixed in `30b86d26` (pushed to `origin/master`). The client's token is retained verbatim on `FPendingCompletion` (`SocketHttpServer.h:136-141`; string or number, both permitted by MCP), forwarded at `SocketHttpServer.cpp:923-924`, stored at `:1007`, returned by the new `GetProgressToken` (`:1081-1092`), and echoed at `PinWrightSubsystem.cpp:435-454` with the ticket id moved into its own `ticket_id` field so a dropped stream still degrades to `system.job_status` polling. **Two further conformance defects in the same 25 lines** were fixed with it: no verb set a progress number, so every frame fell back to the ticket's progress ARRAY LENGTH — ring-trimmed to 50 — and a long job counted to 50 then reported 50 forever against a spec MUST (the wire value now derives from `FJobTicket::ProgressSeq`, which is never trimmed); and `total` was never sent, so no client could draw a bar (now published when the reporter knows its denominator, and omitted rather than zero-filled when it does not). Where the spec and honesty collide, honesty wins and the seam is documented in `rpc-design.md` §9a: MCP says progress MUST increase, but `asset.dump_folder` legitimately sits flat in "waiting for async compilation", so incrementing to satisfy the letter would draw a bar advancing while nothing happened; a reporter that goes backwards is HELD at its previous value, never advanced past it, and the hold is disclosed as `reportedProgress` + `progressHeldAtPrevious`. Verified by `Tests/Transport/TestProgressTokenEcho.cpp` — `StreamRetainsTheClientsToken` asserts the retained token is byte-identical to the client's (`:63-64`) with a failure-direction check (`:67-69`), `PlainRequestRetainsNoToken` asserts none is invented for a plain request (`:122-123`) — and live against a genuinely long operation: 400 assets from disk in 23.64 s, 20 frames carrying progress 0..380 of total 400, **zero client rejections after the fix against 83 before**. Transferable rule recorded at `rpc-design.md:195`: parsing a protocol field and discarding it is worse than not parsing it, because the gate passes and everything downstream looks wired.
