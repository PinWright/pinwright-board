---
id: F-insights-stop-and-trace-path
title: "Complete the `insights.*` namespace: stop_session, get_trace_path, set_channels"
status: DONE
severity: Low
category: feature
tags: [insights, profiling, trace, debug]
---

# Complete the `insights.*` namespace: stop_session, get_trace_path, set_channels

`insights.start_session` exists (`Source/.../Handlers/Debug/InsightsHandler.cpp`, the entire file is one handler that runs `Trace.Start` with an optional `channels` string), but the namespace is otherwise empty. There is no `insights.stop_session`, no `insights.get_trace_path`, and no `insights.set_channels(channels[])`. Confirmed by reading the handler file (28 lines, single `REGISTER_RPC_HANDLER` entry) and grepping `Trace.Stop` / `Trace.SnapshotFile` / `insights.stop` / `insights.get_trace` across `Source/.../Handlers/` — no matches.

Consequence: an agent that calls `insights.start_session` has no MCP-native way to know when the trace stopped, where the `.utrace` landed (`Saved/Profiling/UnrealStats/` vs. `Saved/Profiling/Traces/` vs. wherever `Trace.File` last wrote), or how to mutate the active channel set mid-trace. Traces are effectively dangling — the agent has to either keep the editor running indefinitely or fall back to `python.execute` with raw console commands plus filesystem-access tooling to find the newest `.utrace` by mtime.

Proposed additions, all in `Handlers/Debug/InsightsHandler.cpp`:

- **`insights.stop_session`** — issues `Trace.Stop`. Returns `{ status: "stopped", tracePath: "Saved/Profiling/Traces/Foo.utrace" }`. Resolves the trace path via `FTraceAuxiliary::GetTraceDestination()` (UE 5.4+) or by querying `UE::Trace::ETraceWriteOptions` / falling back to listing the configured trace directory and picking the newest by mtime.
- **`insights.get_trace_path`** — read-only sibling, returns the same `tracePath` field. Useful as a probe before `stop_session` (some workflows want to know the path while the trace is still running).
- **`insights.set_channels`** (params: `channels: string[]` or comma-string) — issues `Trace.Enable`/`Trace.Disable` for the diff against the currently-active set, or simply forwards to `Trace.Channel <name> 1/0`. Lets agents tighten the channel set after seeing initial output without restarting the trace.
- **Optional `insights.snapshot`** — issues `Trace.SnapshotFile` to flush the in-memory ring buffer to disk without stopping. Cheap to add since the handler shape is identical.

Reuse the existing `Ctx.GetString("channels")` parsing pattern from `start_session`. No new state — all four are thin wrappers over `GEngine->Exec` plus a trace-path resolver.

**Workaround:** `python.execute("Trace.Stop")` plus `filesystem-access` `list_dir` on `Saved/Profiling/Traces/` sorted by mtime.

**Fix:** Add the four handlers above to `InsightsHandler.cpp` (the file is currently just `start_session`). Add a private helper `ResolveActiveTracePath()` near the top that returns the destination string via `FTraceAuxiliary` or directory mtime fallback.

## History
- `#1-feature-request` `OPEN` reporter — `insights.start_session` exists alone in `InsightsHandler.cpp` (single 28-line handler); no `stop_session`, `get_trace_path`, `set_channels`, or `snapshot`. Verified by grep on `Trace.Stop`, `Trace.SnapshotFile`, `insights.stop`, `insights.get_trace` across `Source/.../Handlers/` — no matches. Traces are dangling: agents start a session and have no MCP path to know when it ended or where the `.utrace` is. Workaround is `python.execute("Trace.Stop")` + filesystem-access mtime sort. Severity Low — `start_session` alone is still useful, and workarounds exist.
- `#2-implemented-stop-and-trace-path` `IN-REVIEW` developer — Added `insights.stop_session`, `insights.get_trace_path`, `insights.set_channels`, `insights.snapshot` in `Handlers/Debug/InsightsHandler.cpp` via FTraceAuxiliary::Stop/GetTraceDestinationString/WriteSnapshot. Extracted `ResolveActiveTracePath()` into `InsightsHandlerInternal.h` so `Tests/InsightsHandlerTest.cpp` can link directly. Counterfactual: if ResolveActiveTracePath returned a stale/sentinel value, the NotConnectedReturnsEmpty test would fail. No Build.cs change (Core, TraceLog, TraceAnalysis already linked).
- `#3-skip-editor-offline` `SKIP` tester — Editor RPC port 19880 is closed (Invoke-RestMethod refused). Static review of `InsightsHandler.cpp` confirms all four handlers registered (`insights.stop_session`, `insights.get_trace_path`, `insights.set_channels`, `insights.snapshot`) and `InsightsHandlerInternal.h` exports `ResolveActiveTracePath()`, but live verification requires a running editor and the protocol forbids restarting it.
- `#4-verify-fix` `DONE` tester — Verified: `insights.get_trace_path` returns `{tracePath:"", connected:false, connectionType:"None"}` (documented shape) with no active trace. Discovery batch on `insights.stop_session?`, `insights.set_channels?`, `insights.snapshot?` returns full schemas — all four new handlers registered with summaries citing `FTraceAuxiliary::Stop/WriteSnapshot` and `Trace.Enable/Disable`.
