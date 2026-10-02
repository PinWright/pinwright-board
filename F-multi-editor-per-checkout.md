---
id: F-multi-editor-per-checkout
title: "One checkout can host only one MCP-driven editor, and EDITOR_ALREADY_RUNNING names no owner, so two agents on a shared checkout must hand the editor back and forth by hand"
status: IN-REVIEW
severity: Medium
category: feature
tags: [proxy, editor_start, editor_run_tests, editor-lifecycle, port, shared-checkout, multi-agent, launch-reason]
encounters: 1
costly: 1
lastSeen: 2026-09-29T00:00:00Z
---

# One MCP editor slot per checkout

The HTTP port is derived from the project path only
(`UPinWrightSettings::ResolveHttpPort` -> `DerivePortFromPath`, `Source/PinWright/Private/PinWrightSettings.cpp:120-127`;
`:20484` for `X:\src\unreal\unreal-fpv`), and nothing on the command line can move it. So a checkout
has exactly one MCP endpoint. `_editor_process_guard` (`Content/Python/mcp_proxy.py:2439-2478`) refuses
`editor_start` and `editor_run_tests` with `EDITOR_ALREADY_RUNNING` whenever any editor of the checkout
answers on that port, including one that another agent or session launched for its own work (for
example an `UnrealEditor-Cmd ... -ExecCmds="Automation RunTests ..."` run, see
`B-automation-run-binds-mcp-port`).

Observed 2026-09-29 on the PDS checkout `X:\src\unreal\unreal-fpv`: two Claude sessions and their
subagents each needed an MCP-driven editor at the same time. Only one could hold the port; the other
got `EDITOR_ALREADY_RUNNING` and the sessions had to stop, close and relaunch editors to pass the slot
back and forth manually.

The refusal also does not say whose editor it is. The payload is
`{"error": "EDITOR_ALREADY_RUNNING", "url", "port"}` (`mcp_proxy.py:2466-2472`): no pid, no
`launchedBy`, no launch reason, no command line, although `editor_list` already derives all of these
per process (`F-editor-launch-reason-list`). An agent cannot tell whether the blocker is its own
stale editor, a peer's working editor, or a short test run that will exit soon, so it cannot decide
between waiting, asking, or closing it.

**Asked for (either is enough; the owner info is wanted in both):**
1. Per-launch port selection so several MCP editors can serve one checkout: e.g. a
   `-PinWrightPort=<n>` switch (or an ephemeral bind) that `editor_start` / `editor_run_tests` set,
   with the proxy addressing each editor by pid or launch id and `editor_list` reporting each
   editor's `gatewayPort`.
2. Or a lease/queue on the single slot: the refusal carries the owner (pid, `launchedBy`, launch
   reason, start time, mode, command line from the `editor_list` census), and a caller can wait for
   the slot to free instead of polling.

Minimum even without 1 or 2: put the owner fields into the `EDITOR_ALREADY_RUNNING` payload and text.

**Workaround:** coordinate by hand: find the owner with `editor_list`, ask the owning session to
close its editor, then retry.

## History
- `#1-shared-checkout-slot-contention` `OPEN` reporter - Observed on `X:\src\unreal\unreal-fpv` (UE 5.8): two sessions on one checkout could not each drive an MCP editor; `editor_start` refused with `EDITOR_ALREADY_RUNNING` whenever the other side's editor held `:20484`, and the refusal named only the url and port. Sessions handed the slot back and forth manually (editor closes and relaunches). severity rationale: impact=Medium (soft blocker, doable only via manual cross-session coordination and extra editor restarts) x reach=every shared-checkout multi-agent session on this host, no modifier -> Medium. costly=1 (editor restarts spent passing the slot).
- `#2-owner-and-slot-wait` `IN-REVIEW` developer — Implemented ask 2 (wait on the single slot) plus the owner minimum; ask 1 (per-launch port) is not implemented. `EDITOR_ALREADY_RUNNING` from both `_editor_process_guard` (`editor_start` / `editor_restart`) and `_run_tests_guard` (`editor_run_tests`) now carries `owner` = the `editor_list` entry cut to `pid`, `launchedBy`, `reason`, `mode`, `startTime`, `logPath`, `commandLine`, and the text names pid / launchedBy / mode / start / reason. Owner resolution (`Proxy._slot_owner`): the pid the answering editor's `system.identity` reports when it is ready, else the only editor in the census whose checkout publishes the probed port (`gatewayPort`), else `null` with 'owner not identified; editor_list shows every editor of this checkout'. `editor_start` and `editor_run_tests` take `slot_wait` (seconds 0-3600, default 0; else `INVALID_ARGUMENTS`), which polls the port every 5 s until it stops answering, then runs the normal guard, so a caller waits instead of polling; a client disconnect ends the wait. Per-launch ports (`-PinWrightPort`, per-editor port files, proxy routing by pid) were left out: they cross the C++ port resolution / port-file advertisement and the proxy's single-URL model. Files: `Content/Python/mcp_proxy.py` (tool schemas, `_identity_pid`, `_slot_owner`, `_owner_text`, `_wait_for_free_slot`, both guards), `tests/test_mcp_proxy_editor_start.py` (run-tests schema set). Tests: `Content/Python/tests/test_launch_capacity.py` `SlotOwnerTest` (identity pick among two editors, census-only for a starting editor, ambiguous -> null, run-tests refusal), `SlotWaitTest`. Docs: `docs/wiki-src/mcp-transport.md` Guard bullet and tool table, `CHANGELOG.md`.
