---
id: B-editor-pie-premature-success
title: "`editor.play` and `editor.stop` return success when the PIE lifecycle request is only queued for a later tick"
status: OPEN
severity: High
category: bug
tags: [editor, pie, async, lifecycle, false-success]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# PIE lifecycle verbs claim completion at request time

## What happens

`editor.play` calls `RequestPlayInEditorSession` and immediately sends
`success:true` (`PIEHandler.cpp:104-130`). The helper itself states that
`RequestPlaySession` only queues startup for the next tick
(`PieControlUtils.h:63-65`, `:172-173`). No delegate or world-state check confirms
that any requested client, net mode, or emulation session started.

`editor.stop` similarly calls `RequestEndPlayMap` and immediately sends success
(`PIEHandler.cpp:150-159`). The same tree already documents and handles this engine
contract correctly for `editor.quit`: stop is asynchronous and must be polled with
a timeout (`EditorQuitHandler.cpp:74-100`).

## Why it matters

Automation can issue its next input or capture against edit mode after `play`, or
mutate/save while PIE is still tearing down after `stop`. The response gives no
queued state or terminal job to distinguish that race. Severity is High for a
normal-path premature success receipt.

## What should happen

Return a job ticket and complete it only after the requested PIE world contexts are
active, or after no PIE session remains for stop. Use delegates or a bounded ticker
poll, propagate startup/teardown failure, timeout and cancellation, and report the
actual client/net state. If the API intentionally remains fire-and-forget, return
`requested:true`/`queued:true` instead of terminal success.

## Workaround

Poll `editor.pie_status` after either verb and do not continue until the requested
terminal state is observed.

## Related

- `B-editor-quit-crash-pie-active` — contains the existing bounded PIE-stop wait.
- `B-ui-stop-play-exec-noop` — different UI namespace path.

## History
- `#1-source-scan-pie-receipt` `OPEN` reporter — Source-only scan matched both immediate success responses to engine APIs explicitly documented in this checkout as deferred to later ticks. No build, test, editor, MCP call, or plugin edit was performed.
