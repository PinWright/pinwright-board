---
id: B-blocked-on-modal-no-disk-trace
title: "EDITOR_BLOCKED_ON_MODAL was wire-only and never a UE_LOG — a grep over the whole log corpus returned zero whether it had fired once or a thousand times, and a fabricated '779 occurrences' figure survived two triage passes on that silence"
status: DONE
severity: Medium
category: bug
tags: [transport, modal, observability, logging, diagnosability, error-codes, forensics, unfalsifiable-claim]
---

# The one non-retryable failure class left no trace on disk

`EDITOR_BLOCKED_ON_MODAL` existed only on the wire. Both emit sites write it into an HTTP response
body and neither is a `UE_LOG`; the bundled stdio proxy re-emits it to **stderr only**. So the one
failure class that no amount of polling can clear left `Saved/Logs/` completely silent, and its
frequency could be neither confirmed **nor refuted** from disk.

## The two emit sites, and why neither reached a log

Declaration — `Handlers/ErrorCodes.h:262-267`:

```cpp
    // Deliberately NOT a variant of EDITOR_NOT_READY: that code is documented
    // retryable (BuildPingResult sets retryable:true beside it) and a well-behaved
    // client polls it, which is exactly the loop that burns a 600 s watchdog. A
    // modal owning the game thread is not retryable - only a human dismissing the
    // dialog, or killing the process, clears it.
    inline constexpr TCHAR ERR_EDITOR_BLOCKED_ON_MODAL[] = TEXT("EDITOR_BLOCKED_ON_MODAL");
```

Exactly two emissions, both in `Transport/McpRequestCore.cpp`:

- `:391-403` — `BuildPingResult`, sets `error` / `message` / `blockedOnModal` / `blockedSeconds` into the ping result object.
- `:439-445` — `BuildEditorNotReadyToolResult`, returns `MakeToolCallError("[EDITOR_BLOCKED_ON_MODAL] ...")`.

Client side, `Content/Python/mcp_proxy.py` relays it at ten sites, but only through
`log()` — `mcp_proxy.py:311-313`: *"Diagnostics go to stderr ONLY - stdout is reserved for MCP
frames."*

## What the silence cost

A scan reported **"779 `EDITOR_BLOCKED_ON_MODAL` occurrences across five days"** and a P0 modal fix
**"designed and never applied"**. Both are false, and the refutation is now on record:

- The modal fix shipped in `7ff9dcf3` on 2026-08-13, is an ancestor of HEAD, and is intact and
  engaged — `FScopedUnattendedRpc` is entered at `RpcDispatcher.cpp:334` and `:580`, and
  `bSuppressModalDialogsDuringRpc` defaults true (`PinWrightSettings.cpp:33`).
- It is observably working: the only two modals raised in the whole corpus were auto-answered in
  the same millisecond under `GIsRunningUnattendedScript`.
- The 779 never happened. The code appears **0 times in 371 log files**, and its exact wire form
  `[EDITOR_BLOCKED_ON_MODAL]` appears **0 times across every agent transcript** — the only place it
  could ever have landed.

Nothing in the modal path needed fixing. What needed fixing is that none of the above was checkable
by grep. **Two triage passes were spent on a failure class that had never fired**, because a
condition that cannot be counted from disk will be counted from somewhere else.

## Fix (shipped `052977bd`)

`Transport/ModalStateProbe.cpp` now logs the **latch edge**, not the ~60 Hz modal-loop tick, so a
half-hour wedge costs exactly one line and one more when it clears:

- `:112-132` — open: `"Modal window owns the game thread%s%s - every RPC now answers %s (non-retryable) until it is dismissed."`, naming the window title and the wire code.
- `:155-163` — close: `"Modal block cleared after %.1f s%s%s - the game thread is running again."`
  A start line with no end line reads as "the editor never came back", which is a different
  diagnosis from a dialog that was dismissed — the **pair** is what makes the log readable.
- `:15-18` — `DEFINE_LOG_CATEGORY_STATIC(LogPinWrightModal, Log, All)`, its own category so a
  forensic pass greps one name and gets every modal episode with nothing else in the way.

## Verification

`Tests/Transport/TestModalEpisodeLogging.cpp:85-87`,
`PinWright.transport.liveness.Modal.EpisodeLeavesLogTrace`: asserts one line per **episode** rather
than per tick (`:117-118`), that the line names the wire code
(`Capture.CountContaining(ErrorCodes::ERR_EDITOR_BLOCKED_ON_MODAL) == 1`, `:121-122`), and that a
healthy heartbeat stays silent (`:137-138`). Suite reported by the fix commit as 3795 / 3795 / 0.

## The general lesson

This is not a cosmetic gap and not a logging-verbosity preference. **An error code that only ever
exists on the wire is unfalsifiable from disk**: absence of evidence and evidence of absence look
identical, so any number anyone asserts about it survives review. Every non-retryable wire code
should be greppable at its state edge.

## Related

- `B-editor-start-blocks-on-zenserver-modal` — a modal source; now leaves an episode line.
- `B-physics-asset-factory-modal-hang` — likewise.

## History
- `#1-wire-only-code-unfalsifiable` `OPEN` reporter — `EDITOR_BLOCKED_ON_MODAL` is emitted at exactly two sites, `Transport/McpRequestCore.cpp:391-403` (`BuildPingResult`) and `:439-445` (`BuildEditorNotReadyToolResult`); both write into an HTTP response body and neither is a `UE_LOG`, and the bundled stdio proxy relays it to stderr only (`Content/Python/mcp_proxy.py:311-313`). A grep over the whole log corpus therefore returns zero whether the condition fired once or a thousand times. The concrete cost is on record: a scan asserted "779 EDITOR_BLOCKED_ON_MODAL occurrences across five days" plus a P0 modal fix "designed and never applied", and **both claims are false** — the modal fix shipped in `7ff9dcf3` on 2026-08-13, is an ancestor of HEAD, `FScopedUnattendedRpc` is entered at `RpcDispatcher.cpp:334` and `:580`, `bSuppressModalDialogsDuringRpc` defaults true (`PinWrightSettings.cpp:33`), the only two modals in the whole corpus were auto-answered in the same millisecond under `GIsRunningUnattendedScript`, the code appears 0 times in 371 log files, and its wire form `[EDITOR_BLOCKED_ON_MODAL]` appears 0 times in every agent transcript. Two triage passes were spent on a failure class that had never fired. Nothing in the modal path is defective; the defect is that none of this was checkable by grep.
- `#2-episode-trace-added` `IN-REVIEW` developer — Fixed in `052977bd`. `Transport/ModalStateProbe.cpp` logs the latch EDGE rather than the ~60 Hz modal-loop tick, so a half-hour wedge costs one line and one more on clear: `:112-132` names the window title and the wire code and states the code is non-retryable; `:155-163` reports the duration on clear, so a start with no end reads as "the editor never came back" — a different diagnosis from a dismissed dialog. `LogPinWrightModal` is its own category (`:15-18`) so a forensic pass greps one name. The wire behaviour is unchanged; this adds only the disk trace that makes claims about it falsifiable.
- `#3-verified` `DONE` tester — `PinWright.transport.liveness.Modal.EpisodeLeavesLogTrace` (`Tests/Transport/TestModalEpisodeLogging.cpp:85-87`) asserts one line per episode rather than per tick (`:117-118`), that the emitted line names the wire code (`:121-122`), and that a healthy heartbeat stays silent (`:137-138`) — so the test fails both on a missing trace and on a per-tick flood. Suite reported by the fix commit as **3795 / 3795 / 0**. Filed retroactively. The transferable rule, worth more than the fix: an error code that exists only on the wire is unfalsifiable from disk, because absence of evidence and evidence of absence look identical — which is exactly how a fabricated count survived two reviews.
