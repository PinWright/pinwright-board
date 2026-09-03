---
id: B-port-in-use-status-unreachable
title: "EMcpServerStatus::PortInUse is unreachable — a failed bind reports Disabled, so the conflict-gated setup screen and its red banner are dead code"
status: OPEN
severity: Medium
category: bug
tags: [transport, http, bind, port-conflict, status, setup-screen, dead-code, regression]
encounters: 1
lastSeen: 2026-08-18T02:21:06Z
---

# `PortInUse` can never be returned, so both of its consumers are dead

`UPinWrightSubsystem::GetServerStatus()` (`PinWrightSubsystem.cpp:273-283`) has exactly two
outcomes: `Disabled` when the transport pointer is invalid, otherwise
`bStreamingActive ? Listening : Disabled`. `EMcpServerStatus::PortInUse` is never returned by any
code path. The source says so itself, in the comment immediately above the return:

> *The socket server has no port-conflict introspection; a failed bind reports Disabled rather
> than PortInUse.*

The enum still declares the meaning that is being dropped —
`Public/PinWrightSubsystem.h:27`: *"the configured port is held by another process; this editor is
not serving"*.

## Both consumers are unreachable branches

- **`PinWrightModule.cpp:226-228`** —
  `bProblemAlert = ((Status == EMcpServerStatus::PortInUse) || bAnyInstalledConfigOutdated) && Settings->bShowSetupScreenOnProblem`.
  The port-conflict half never fires, so the setup screen auto-opens on a stale agent config only.
  A conflicted editor — the case the alert was built for — starts silently.
- **`Setup/SGatewaySetupScreen.cpp:172`** — the `if (Status == EMcpServerStatus::PortInUse)` banner
  branch is unreachable, so a conflicted editor renders the amber *disabled* state instead of the
  red *conflict* state. That is also a wrong readback in its own right: MCP is enabled in
  settings, and the reason it is not serving is a bind failure, not a disable.

The information exists at the point of failure and is thrown away before anyone asks for it.
`Saved/Logs/DroneFootball_2.log:2741` from this session:

```
LogSocketHttp: Error: Port 24281 could not be bound - it is either already in use (e.g. another
project's editor on the same default port), or reserved by the OS ... This editor's MCP server is
NOT serving.
```

The socket layer knows precisely what happened; `GetServerStatus()` reports `Disabled`.

## This is a regression, and it sits under an IN-REVIEW ticket

`B-port-conflict-false-active` (**IN-REVIEW**, High) shipped this exact mechanism: history entry
`#2-preflight-probe-and-status` records a pre-flight loopback bind probe plus
`bPortConflict` / `DidPortConflict()` on `FMcpTransport`, `GetServerStatus()` + `EMcpServerStatus`
on the subsystem, and the setup-screen status banner with conflict-gated auto-open. The
`FMcpTransport` → `FSocketHttpServer` rewrite replaced the transport with one that reports only a
`bool` from `Start()`, and the conflict wire went with it — leaving the enum, the banner branch
and the module gate behind as dead code.

**That ticket is awaiting a tester.** A tester who verifies only the refuse-to-serve half will
find it genuinely works (`SocketHttpServer.cpp:264+` fails the bind and the editor does not serve,
confirmed by the log above) and close it, while the status-and-UI half of its own remedy is gone
from current source. Recorded here rather than by returning `B-port-conflict-false-active` to
`OPEN`, because its refuse-to-serve half is real and a return would discard that verdict — but the
tester should read this ticket before setting `DONE`.

**Fix:** `FSocketHttpServer::Start` already isolates the bind failure from its other failure modes
(`SocketHttpServer.cpp:264`, distinct from the socket-subsystem and socket-create branches at
`:244-258`). Expose it — a `DidPortConflict()` accessor mirroring the one
`B-port-conflict-false-active` added to `FMcpTransport` — and return `PortInUse` from
`GetServerStatus()` on that path. Add a regression test asserting that a second bind against a
held port yields `PortInUse`, not `Disabled`; the enum's only coverage today is via a UI branch,
which is exactly why the rewrite dropped it without a single test going red.

## Related

- `B-port-conflict-false-active` (IN-REVIEW) — shipped this mechanism; the transport rewrite
  regressed it. Read before closing that ticket.
- `B-failed-bind-no-retry` (OPEN) — the same failed bind seen as a lifecycle defect: one attempt,
  then a live non-serving editor. Same log lines, different remedy.
- `B-no-editor-identity-handshake` (OPEN) — the client-side half of the multi-editor problem.

## History
- `#1-dead-conflict-branch` `OPEN` reporter — `GetServerStatus()` (`PinWrightSubsystem.cpp:273-283`) returns only `Disabled` or `Listening`; `EMcpServerStatus::PortInUse` is unreachable, as the in-source comment at `:279-280` states outright ("The socket server has no port-conflict introspection; a failed bind reports Disabled rather than PortInUse"), while `Public/PinWrightSubsystem.h:27` still declares its meaning. Both consumers are therefore dead: the conflict-gated setup-screen auto-open at `PinWrightModule.cpp:226-228` (`bProblemAlert` fires on a stale agent config only, never on a port conflict) and the red conflict banner at `Setup/SGatewaySetupScreen.cpp:172` (a conflicted editor renders amber "disabled" instead — itself a wrong readback, since MCP is enabled and the bind is what failed). The truth exists one layer down and is discarded: `Saved/Logs/DroneFootball_2.log:2741` shows `LogSocketHttp` naming the exact condition ("Port 24281 could not be bound - it is either already in use ... This editor's MCP server is NOT serving"). **Regression, not an original gap:** `B-port-conflict-false-active` (IN-REVIEW) shipped the probe + `bPortConflict`/`DidPortConflict()` + `GetServerStatus()` + banner (its `#2-preflight-probe-and-status`), and the `FMcpTransport` → `FSocketHttpServer` rewrite dropped the conflict wire because the new transport reports only a bool. Flagged rather than returning that ticket to OPEN — its refuse-to-serve half genuinely works — but a tester must not close it on the refuse-to-serve verification alone. Fix: expose the isolated bind-failure branch (`SocketHttpServer.cpp:264`) through a `DidPortConflict()` accessor and return `PortInUse`, with a test asserting a second bind on a held port reports `PortInUse` rather than `Disabled` — the enum's only current coverage is a UI branch, which is why the rewrite regressed it with no test going red. Remedy assigned this session to the `wave-1-pinwright-transport` chunk of the arena-rescaling plan; not yet implemented at filing.
