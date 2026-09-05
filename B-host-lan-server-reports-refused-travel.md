---
id: B-host-lan-server-reports-refused-travel
title: "session.host_lan_server reports travelExecuted true even when UWorld::ServerTravel rejects or does not schedule the requested travel"
status: IN-REVIEW
severity: High
category: bug
tags: [session, server-travel, false-success, readback, multiplayer]
encounters: 1
lastSeen: 2026-09-03T23:17:31+03:00
---

# LAN host setup ignores the server-travel outcome

## What happens

With `executeTravel:true`, `session.host_lan_server` resolves a world and calls
`World->ServerTravel(TravelURL, true)` but drops its boolean result
(`Handlers/System/SessionsHandler.cpp:188-201`). It leaves `bSuccess` true, returns status
`"server travel initiated"`, and sets `travelExecuted:true` at `:210-220` solely because a world
pointer existed.

UE 5.8 returns false when `GameMode->CanServerTravel` refuses the URL
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/World.cpp:9525-9532`). It also schedules the URL
only when no level change is already pending and seamless-travel conditions permit it at
`:9537-9555`, yet returns true at `:9557` even when that block did not run. The handler therefore
has both a definite rejected-call false success and an already-travelling no-schedule false success.

## Why it matters

A caller proceeds as if a listen server is moving to the requested map, while clients remain on
the old/pending world. The response's strongest field, `travelExecuted`, is the lie. Severity is
High on the normal LAN-host path.

## What should happen

Capture and enforce `ServerTravel`'s return value. Also distinguish “accepted/scheduled” from
“completed”: compare the post-call pending travel state with the requested URL, refuse when an
existing travel prevented scheduling, and expose a queued/accepted field. Do not claim execution
until an observable world/map transition occurs; direct callers to `editor.pie_status` for that
terminal readback or model the transition as a job.

**Workaround:** Poll `editor.pie_status` and verify the server world's `mapName` after the call.

## Related

- Catalog: `request-echo-not-result-readback`, `premature-async-success`, `terminal-success-before-completion-or-invariant`
- `F-pie-status-rpc` — provides the independent world-state readback needed for a workaround.
- `E-session-wiki-pie-prerequisite-undocumented` — prerequisite documentation, not travel outcome handling.

## Fix

The root cause was that the handler discarded `ServerTravel`'s boolean result and treated world
presence as both scheduling and completion. It now evaluates the call result together with the
pre/post pending URL and seamless-transition state, returns typed `TRAVEL_REFUSED` unless this
request newly queued its destination, and uses the retained bounded PIE lifecycle waiter to
re-resolve the relevant world until the requested map is observed with no pending transition.
`travelExecuted` now mirrors only that completed readback.

Exact files changed:
- `Source/PinWright/Private/Handlers/System/SessionsHandler.cpp`
- `Source/PinWright/Private/Handlers/System/SessionsTravelDecision.h`
- `Source/PinWright/Private/Handlers/ErrorCodes.h`
- `Source/PinWright/Private/Tests/EditorOps/TestSessionTravelOutcome.cpp`
- `Docs/wiki-src/session.md`
- `Docs/wiki-src/editor.md` — reciprocal cross-link to the session hosting contract
- `X:/src/unreal/.pinwright-board/B-host-lan-server-reports-refused-travel.md`

Regression coverage: `FSessionHostLanServerTravelOutcomeContractTest`, test id
`PinWright.session.host_lan_server.TravelOutcomeContract`.

Deliberate non-changes: no new ticker, no changes to `Utils/PieState.*`, no engine, Config, Saved,
or outer PDS edits, and no build, editor, MCP, or live travel run in this source-only pass.

## History
- `#1-servertravel-result-ignored` `OPEN` reporter — Source-read the handler and UE 5.8
  `UWorld::ServerTravel`; confirmed the response ignores both the false return and the
  no-new-schedule branch. No PIE or travel was started.
- `#2-travel-outcome-contract` `IN-REVIEW` developer — Captured `ServerTravel` plus pre/post
  pending state in `SessionsHandler.cpp`, added typed refusal and bounded destination readback,
  documented the accepted/queued/completed contract, and added
  `PinWright.session.host_lan_server.TravelOutcomeContract`.
