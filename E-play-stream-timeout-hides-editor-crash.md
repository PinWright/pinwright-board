---
id: E-play-stream-timeout-hides-editor-crash
title: "When the editor crashes during editor.play, the proxy reports 'stream ended early (timed out) ... may still be running, poll system.job_status' instead of saying the editor process died"
status: OPEN
severity: Low
category: ergonomic
tags: [editor.play, mcp-proxy, crash, error-message, stream, job-status]
encounters: 1
lastSeen: 2026-09-28T10:27:00Z
rice: [1, 2, 1, 2]
priority: 8
---

# A dead editor is reported as a slow operation

UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv-wt1` (Linux), plugin `61c243f5`. `editor.stop` then
`editor.play {}` in the same editor. The game crashed with SIGSEGV inside PIE BeginPlay
(`UDroneSelectionSubsystem::GetDroneForController`, a game bug caused by assets changing on disk under
the running editor). The call returned:

```
Editor stream at http://127.0.0.1:27673/mcp ended early (stream read failed: timed out).
The operation may still be running in the editor - poll it with system.job_status.
```

The next call returned `EDITOR_UNRESPONSIVE ... retry later and do not start a second editor`. The
editor was already in `StaticShutdownAfterError` with CrashReportClientEditor up; nothing would ever
answer `system.job_status`. The advice to wait and not start another editor is wrong in this state.

**Fix (proposed):** when a forwarded call's stream fails, have the proxy check whether the editor pid it
knows (or the pid owning the gateway port) is still alive and not stopped, and scan the tail of
`Saved/Logs/<Project>.log` for `=== Critical error: ===`; if either shows a crash, return a distinct
`EDITOR_CRASHED` error naming the log path and the first frame.

## History
- `#1-crash-reported-as-timeout` `OPEN` reporter - Seen during a PDS QA verification. Cost: ~2 min and a log read to learn the editor had crashed; no work lost.
