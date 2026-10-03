---
id: B-editor-start-guard-ignores-booting-editor-process
title: "editor_start's single-editor guard only probes the MCP port, so a retry while the first editor is still booting (port not yet bound) launches a second editor of the same checkout"
status: OPEN
severity: Low
category: bug
tags: [proxy, mcp-proxy, editor-launch, editor_start, guard, cold-start]
encounters: 1
costly: 0
lastSeen: 2026-10-02T00:00:00Z
rice: [1, 2, 1, 1]
priority: 17
---

# The launch guard cannot see an editor that has not bound its port yet

`_editor_process_guard` (`Content/Python/mcp_proxy.py`, called by `editor_start`,
`editor_restart`'s start half and `editor_run_tests`) decides from
`_editor_process_observation`, which is `_probe_state` on the checkout's MCP URL and
nothing else. `not_running` (connection refused) clears the spawn. An editor of this
checkout that is alive but still early in boot - before the PinWright transport binds
the port - answers exactly that, so the guard lets a second editor of the same
checkout start. Only one can bind the port; the other boots into a lost bind and
serves nothing.

The process census that would see the first editor already exists: `editor_list`
enumerates every Unreal editor on the machine with its checkout, pid, `launchedBy`
and `-Abslog`.

The window is real now that `EDITOR_START_TIMEOUT` can return with
`probeState: "not_running"` (process alive, port unbound): the timeout text tells the
caller not to call `editor_start` again, but nothing enforces it.

## Fix direction
When the port probe says `not_running`, also consult the `editor_list` census for a
live editor process of this checkout (same `.uproject`), and refuse with
`EDITOR_ALREADY_RUNNING` (owner from the census) when one exists - or at least when
one was launched by PinWright and is younger than the readiness ceiling.

## History
- `#1-found-in-review` `OPEN` developer - Found during review of `F-editor-start-readiness-timeout-override` (`#5-per-call-timeout`): with a per-call `timeout`, `EDITOR_START_TIMEOUT` reports `probeState: not_running` for a booting editor whose port is not bound yet, and a retry of `editor_start` at that point passes `_editor_process_guard`, which reads only `_probe_state` (connection refused = clear) and never the process list. severity rationale: impact=Low - requires a caller to ignore the timeout text's explicit "do not call editor_start again", and the result is a second editor that loses the bind, not data loss; reach=slow cold boots only.
