---
id: B-failed-bind-no-retry
title: "A failed port bind is terminal: the editor stays live and non-serving for its whole lifetime with no retry, while the gateway-port file keeps advertising an endpoint it does not serve"
status: IN-REVIEW
severity: High
category: bug
tags: [transport, http, bind, retry, gateway-port, orphan, multi-editor, startup, sse]
encounters: 1
lastSeen: 2026-08-18T02:21:06Z
---

# A failed bind is terminal — one attempt, then a live editor with no MCP surface

`UPinWrightSubsystem` calls `StreamingTransport->Start(EffectivePort)` **exactly once**
(`PinWrightSubsystem.cpp:181`). On failure it logs one Error (`:196-199`), leaves
`bStreamingActive` false, and continues initializing. Nothing ever calls `Start` again. The
editor is otherwise completely alive — Slate, the core ticker, Python, Live Coding — but it has
no MCP surface for the rest of the process lifetime, and the only remedy available to a user is
a full editor restart.

## Session evidence

`Saved/Logs/DroneFootball_2.log:2740-2742`:

```
[2026.08.18-02.21.06:059] LogPinWrightSubsystem: HTTP transport binding port 24281.
[2026.08.18-02.21.06:059] LogSocketHttp: Error: Port 24281 could not be bound - it is either
    already in use (e.g. another project's editor on the same default port), or reserved by the
    OS ... This editor's MCP server is NOT serving.
[2026.08.18-02.21.06:059] LogPinWrightSubsystem: Error: Transport failed to bind port 24281.
```

`grep -n "binding port"` over the whole 5,793-line log returns **one** hit: the single attempt at
`02:21:06`. The log's last entry is timestamped `06:36:29` — **4 h 15 min** of a live editor with a
dead automation surface, during which no retry was attempted and nothing re-checked the port.
`:2743` then logs `PinWrightSubsystem initialized (socket transport).` — the same success-shaped
line a serving editor writes.

## The retry is cheap and every piece of it already exists

- `FSocketHttpServer::Start` early-returns `true` when already active
  (`Transport/SocketHttpServer.cpp:229-232`) and leaves a clean re-`Start()`-able state on every
  failure path — missing socket subsystem (`:244-248`), socket-create failure (`:254-258`), and
  the bind itself (`:264+`). Calling it again is safe and idempotent.
- The 0.1 s core ticker is registered **unconditionally** at `PinWrightSubsystem.cpp:209-212`,
  after the transport block, so it runs even when the bind failed. There is already a timer to
  hang a bounded retry on.
- `Tests/Transport/SocketTestClient.h` already exercises repeated `Start()`.

## Second half: the port file keeps pointing somewhere else

`GatewayPortFile::WritePortFile` is called **only** inside the success branch
(`PinWrightSubsystem.cpp:189`), with the in-source rationale *"Written only on a successful bind so
the last-known-good file survives failures."* After a failed bind,
`<Project>/Saved/PinWright/gateway-port` still names whatever the last successful bind wrote —
possibly a previous editor's port that nothing is serving any more.

`mcp_proxy.py` re-resolves the endpoint from that file **on every call**
(`Content/Python/mcp_proxy.py:1007-1028`) — exactly right for a live rebind, and exactly wrong
here: the client is confidently pointed at an endpoint the live editor does not serve, and
nothing on either side says so. That last-known-good policy is defensible on its own; it is only
harmful in combination with never retrying, which is why both halves belong in one ticket.

Two more things stay silently dead in the failure branch: `StreamingPort` is never set, so
`GetBoundHttpPort()` returns 0 (`:268-271`), and `JobEventHandle` — the `FJobRegistry::OnJobEvent`
subscription that bridges job events onto open SSE streams — is subscribed only on success
(`:192-193`). Any late rebind that forgets it would serve RPCs normally while every SSE progress
frame stayed dead.

## Related — deliberately not folded into `B-no-editor-identity-handshake`

`B-no-editor-identity-handshake` (OPEN, High) describes the same wreck from the other side of the
wire, and its **Fix** line already names half of this one: *"and/or have the failed-bind editor
exit (or hard-disable MCP-dependent automation) rather than orphan as a live non-serving
process."* Its subject, though, is the **client's** inability to tell which editor answered it,
and its evidence is a misrouted mutating RPC that crashed the user's editor. Filed separately
because the remedies are independent and both are required: a bounded rebind on the failing
editor (here) re-introduces precisely the misroute hazard that ticket describes unless the caller
can also verify who answered after the rebind (there).

**Fix:** bounded retry driven from the existing core ticker — roughly 60 s with backoff, then stop
and log once at Error. A late success must do everything the success branch does today
(`:186-193`): set `StreamingPort`, `GatewayPortFile::WritePortFile`, **and** subscribe
`JobEventHandle`; skipping the last leaves a server whose progress stream is silently dead. Pair
it with an identity RPC (`B-no-editor-identity-handshake`) so a caller can confirm which editor
picked up the port after a rebind. `mcp_proxy.py` needs no change — it already re-resolves the
port per call for exactly this scenario.

## History
- `#1-one-attempt-then-orphan` `OPEN` reporter — `UPinWrightSubsystem` calls `StreamingTransport->Start()` once (`PinWrightSubsystem.cpp:181`); on failure it logs an Error (`:196-199`), leaves `bStreamingActive` false and continues, so the editor stays fully alive with no MCP surface until it is restarted. Session evidence `Saved/Logs/DroneFootball_2.log:2740-2742` (`HTTP transport binding port 24281.` → `LogSocketHttp: Error: Port 24281 could not be bound ...` → `Transport failed to bind port 24281.`): exactly one `binding port` line in the whole 5,793-line log, attempt at `02:21:06`, last log activity at `06:36:29` — **4 h 15 min** unreachable with no retry, and `:2743` still logging the ordinary `PinWrightSubsystem initialized` line. A retry is cheap: `FSocketHttpServer::Start` early-returns true when active (`SocketHttpServer.cpp:229-232`) and leaves a re-`Start()`-able state on all three failure paths, and the 0.1 s core ticker is registered unconditionally at `PinWrightSubsystem.cpp:209-212` so it runs even after a failed bind. Second half: `GatewayPortFile::WritePortFile` runs only on success (`:189`, "last-known-good file survives failures"), so `Saved/PinWright/gateway-port` keeps naming an endpoint this editor does not serve, and `mcp_proxy.py:1007-1028` re-resolves from that file per call — pointing the client at a dead endpoint with no signal. Also dead in the failure branch: `StreamingPort` (so `GetBoundHttpPort()` returns 0, `:268-271`) and the `JobEventHandle` SSE bridge (`:192-193`), which a late rebind must re-establish or progress frames stay silently dead. Deliberately **not** appended to `B-no-editor-identity-handshake` (OPEN) even though that ticket's Fix line already names the orphan half: its subject is the client-side inability to identify which editor answered, and the two remedies are independent — a late rebind without an identity handshake re-introduces exactly the misroute that ticket documents. Remedy assigned this session to the `wave-1-pinwright-transport` chunk of the arena-rescaling plan (bounded retry + `system.identity`); not yet implemented at filing.
- `#2-bounded-retry-stale-port` `OPEN` reporter — Additional evidence: **Adversarial review A — REFRAME.** Actuality: PARTIAL. Framing: the one-shot bind/orphan is no longer current in this working tree: `UPinWrightSubsystem::Initialize` uses `FPortBindRetry::TryBind`, arms a 60-second backoff campaign, and `Tick` publishes a late success; however these source changes are uncommitted and the stale-advertisement half survives exhaustion. `GetServerStatus()` remains `PortInUse`, while `mcp_proxy.py` still follows the last-known-good port. Proposed fix: INCOMPLETE, the shared retry/publish path is systemic and covers bound state, gateway-port, and SSE subscription, but `TryBind` ignores `GatewayPortFile::WritePortFile` failure and never invalidates the stale file after budget exhaustion; `system.identity` is advisory, not an enforced target check. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\PinWrightSubsystem.cpp:173-200,317-327`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Transport\PortBindRetry.cpp:38-77`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\GatewayPortFile.cpp:52-79`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Transport\TestPortBindRetry.cpp:139-161,166-230`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:1006-1030`, `C:\UE_5.8\Engine\Source\Runtime\Sockets\Public\Sockets.h:103-125`. Runtime: NOT VERIFIED (no Unreal build/editor run). Recommendation: REFRAME; keep OPEN for stale-file/publication-failure behavior, make exhaustion invalidate or quarantine the port file and handle write errors, then run focused transport tests plus a two-editor/rebind/identity acceptance check before changing status.
- `#3-owner-aware-handoff` `OPEN` reporter — Additional evidence: **Adversarial review B — RETAIN WITH REFRAME.** Actuality: PARTIAL. Framing: A is right that the one-shot/orphan path is gone in the dirty plugin tree, but the original title is stale; the remaining endpoint-publication and handoff ambiguity can still block or misroute mutations, so High remains defensible. I disagree with blanket invalidation/quarantine at exhaustion: `gateway-port` is deliberately last-known-good, and a failed second editor can coexist with a first editor legitimately serving that same port; deleting it could break the real owner. Proposed fix: INCOMPLETE, the shared retry/publish path covers late bound state and SSE, but `TryBind` returns success when `GatewayPortFile::WritePortFile` fails and `system.identity` has no proxy/server enforcement; a one-time identity check also races a late port takeover. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\PinWrightSubsystem.cpp:182-199,293-324`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Transport\PortBindRetry.cpp:9-25,45-75`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\GatewayPortFile.cpp:52-81`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Content\Python\mcp_proxy.py:1006-1030,2241-2300`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\System\IdentityHandler.cpp:27-58`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Transport\TestPortBindRetry.cpp:54-163,171-230`. Runtime: NOT VERIFIED (the retry, identity, and tests are uncommitted/untracked in the dirty plugin worktree; no Unreal build/editor/test run). Recommendation: REFRAME; keep OPEN, make publication owner/generation-aware (or lease/lock based), treat write failure transactionally, and add two-editor handoff, live/dead last-known-good, publication-failure, and identity-to-mutation race tests before changing status.
- `#4-advertisement-retracted-on-proof` `IN-REVIEW` implementer — Both halves are now implemented in the EAContentExamples58 plugin checkout. Retry half was already in at `a7ef38f2` (`Public/Transport/BindRetryPolicy.h`, `UPinWrightSubsystem::TryStartTransport` / `TickBindRetry`, 2 s doubling to 30 s cap, 600 s budget, 300 s terminal re-log, `EMcpServerStatus::PortInUse` reaching the setup screen's red banner) — that answers review A's "no longer current" reading of the one-shot orphan. This session closes the stale-advertisement half both reviews kept open, at `d8609ec2` + `f86d12c3`. New `Private/Transport/PortAdvertisement.h/.cpp`: `Judge(bHasClaim, bSomethingListening)` is a pure verdict (`NoClaim` / `Retain` / `Retract`), the single impure input is a non-blocking loopback connect with a 0.25 s budget whose every inconclusive outcome reads as *listening*. `ReconcileWhileNotServing()` is called from every not-serving path in `TryStartTransport` (first collision failure, terminal non-collision failure) and again on the exhausted re-log cadence, so a holder that exits after we gave up still turns a Retain into a Retract. **This deliberately implements review B's position over review A's, and the disagreement is resolved in B's favour:** no blanket invalidation or quarantine at exhaustion. A second editor of the same project loses the bind precisely because the first holds that port and is serving on it, so the claim is retracted only on proof it is false — ownership is unknowable after a crash, liveness is observable. Review A's other finding is fixed as stated: `WritePortFile`'s return value is no longer discarded — `UPinWrightSubsystem::PublishBoundPort` logs an Error naming the path on failure, sets `bPortPublishPending`, and retries every tick, because a bound server no client can discover is indistinguishable in the log from one that never bound. `GatewayPortFile` gained `ReadPortFile` (rejects empty / non-numeric / out-of-range / non-canonical, accepts surrounding whitespace) and `RemovePortFile`. `mcp_proxy.py` unchanged, as the ticket predicted. **Not addressed here and still open elsewhere:** `system.identity` enforcement and the identity-to-mutation race are `B-no-editor-identity-handshake`'s, per this ticket's own Related section. **Tests** (+5, `Tests/Transport/TestPortAdvertisement.cpp`): `PinWright.transport.port_advertisement.` `VerdictRetractsOnlyWhatIsProvablyDead`, `ProbeSeparatesAListenerFromAnEmptyPort`, `RetractsAnAdvertisementNothingIsServing`, `RetainsAnAdvertisementSomethingIsServing`, `UnreadableClaimIsNoClaim`. Failure direction runs **both** ways on purpose: `RetractsAn...` fails if a dead claim is left standing, `RetainsAn...` fails if the reconcile is ever "simplified" into an unconditional delete — either test alone is satisfied by a one-line change that recreates the other defect. Real loopback sockets, no stubs; ephemeral bind-then-release supplies a provably unserved port. **Runtime: NOT VERIFIED end to end.** All five touched TUs compile clean (`-SingleFile -NoHotReloadFromIDE`, real `[1/1] Compile [x64]` lines, `Result: Succeeded`); the suite was not run by this session by instruction, and no two-editor contention run was performed. Docs same commit: `Docs/rpc-design.md` §19 (honest failure: a service that cannot serve must say so loudly and must retract what it advertised), `Docs/wiki-src/unattended.md`, `Docs/wiki-src/mcp-transport.md`, `Docs/arch.md`.
