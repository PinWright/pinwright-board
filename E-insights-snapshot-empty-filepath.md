---
id: E-insights-snapshot-empty-filepath
title: "insights.snapshot reports filePath:\"\" for auto-generated snapshots, hiding where the .utrace landed"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [insights, profiling, trace, snapshot, result-misreport]
---

# insights.snapshot reports `filePath:""` for auto-generated snapshots

`insights.snapshot` with `filePath` omitted writes a `.utrace` to disk (the
engine auto-generates the name via `FTraceAuxiliary::WriteSnapshot(nullptr)`)
and correctly reports `status:"snapshot_written"` — but the `filePath` field
in the response is the **empty string**, not the path the engine actually
wrote. The handler echoes back the *input* argument verbatim instead of the
*resolved* destination, so when the caller relies on the documented
auto-generate behaviour the response gives no way to learn where the snapshot
landed. The result object is internally self-contradictory: it simultaneously
claims the snapshot was written and reports an empty path for it.

The asymmetry is the giveaway — supply `filePath` explicitly and it round-trips
fine; omit it (the exact case the param doc invites: *"if omitted, the engine
auto-generates one"*) and the path silently vanishes from the response. Every
sibling in the namespace that produces a path resolves and returns the real
one: `insights.stop_session` and `insights.get_trace_path` both return the
resolved `tracePath` via `ResolveActiveTracePath()` /
`FTraceAuxiliary::GetTraceDestinationString()`. Only `snapshot` reports a path
it never resolved.

Root cause (`Source/.../Handlers/Debug/InsightsHandler.cpp`, the
`insights.snapshot` handler):

```cpp
const FString FilePath = Ctx.GetString(TEXT("filePath"));
const bool bOk = FTraceAuxiliary::WriteSnapshot(FilePath.IsEmpty() ? nullptr : *FilePath);
...
Result->SetStringField(TEXT("filePath"), FilePath);   // echoes the (empty) INPUT, not the written path
```

When `FilePath` is empty the engine picks the name itself, but the handler
writes the still-empty input back into the `filePath` result field.

## Verbatim repro (live, replay-confirmed)

1. `insights.start_session` `{channels:"cpu,frame"}` → `{"channels":"cpu,frame","action":"start_trace","status":"started"}`
2. `insights.get_trace_path` `{}` → `{"tracePath":"X:/.../Saved/Profiling/20260617_144541_1B7740.utrace","connected":true,"connectionType":"File"}` (trace is active)
3. `insights.snapshot` `{}` (no filePath) → **`{"status":"snapshot_written","filePath":""}`**
   - On disk, a new `Saved/Profiling/20260617_144550_5E4CA0.utrace` (34 MB) appeared at exactly that moment — the snapshot *was* written, the response just doesn't say where.
4. Contrast, `insights.snapshot` `{filePath:"X:/.../Saved/Profiling/replay_explicit.utrace"}` → `{"status":"snapshot_written","filePath":"X:/.../Saved/Profiling/replay_explicit.utrace"}` (explicit path round-trips correctly).

So `status:"snapshot_written"` + `filePath:""` from the same call is the
quotable misreport: the path is empty precisely in the documented
auto-generate path.

**Impact:** an agent that snapshots without specifying a path (e.g. "flush a
durable .utrace mid-trace") cannot programmatically locate the file from the
response — it has to fall back to mtime-sorting `Saved/Profiling/` on disk,
which is exactly the friction the `insights.*` namespace was completed to
remove. Low severity (explicit-path callers and `stop_session` are unaffected;
the file is not lost, just unreported).

**Workaround:** always pass an explicit `filePath` to `insights.snapshot`; or
after an omitted-path snapshot, mtime-sort `Saved/Profiling/*.utrace` to find
the newest file.

**Fix:** when `FilePath` is empty, resolve and return the actual written path
instead of the empty input. `FTraceAuxiliary::WriteSnapshot` returns only a
`bool`, but the engine exposes the resolved destination through the public
`FTraceAuxiliary::OnSnapshotSaved` multicast delegate
(`Engine/Source/Runtime/Core/Public/ProfilingDebugging/TraceAuxiliary.h`), which
it **broadcasts synchronously inside the `WriteSnapshot` call** with the
fully-resolved native path — including the auto-generated name when the input is
null/empty (`TraceAuxiliary.cpp`, `WriteSnapshot` →
`OnSnapshotSaved.Broadcast(EConnectionType::File, NativePath)`). So bind a
temporary lambda to `OnSnapshotSaved` immediately before the write, capture
`TraceDestination`, unbind right after, and set `filePath` to the captured path.
This is deterministic and race-free — do **not** mtime-sort the profiling
directory for the newest `.utrace` (a concurrent capture can win that race, the
same correction the sibling `E-memory-report-no-path` #4 had to make), and do
**not** reuse `ResolveActiveTracePath()` / `GetTraceDestinationString()` (that
returns the *active session's* trace destination, a different file from the
snapshot's separately-timestamped one — the repro's session `…144541` vs.
snapshot `…144550` prove they differ). If the delegate yields nothing, fall back
to the (non-empty only for explicit) requested path and flag `pathResolved:false`
so the response stops contradicting its own `snapshot_written` status.

## History
- `#1-initial-repro` `OPEN` reporter — `insights.snapshot {}` (no `filePath`) returns `{"status":"snapshot_written","filePath":""}` while writing an engine-auto-named `.utrace` to `Saved/Profiling/` (confirmed: `20260617_144550_5E4CA0.utrace`, 34 MB, appeared at call time). Handler echoes the empty input arg into the `filePath` result instead of the resolved destination (`InsightsHandler.cpp`, `Result->SetStringField("filePath", FilePath)` after `WriteSnapshot(FilePath.IsEmpty() ? nullptr : ...)`). Asymmetry confirmed: explicit `filePath` round-trips correctly; omitted `filePath` — the documented auto-generate path — drops it. Sibling RPCs (`stop_session`, `get_trace_path`) resolve and return the real path via `ResolveActiveTracePath()`; only `snapshot` doesn't. Replay-confirmed live against `mcp__editor-automation__call`. Ergonomic: result misreports the path field; the snapshot file is written but unreported.
- `#2-additional-replay` `OPEN` reporter — Additional evidence (independent session, seed `insights.snapshot`): re-confirmed live via `mcp__editor-automation__call`. `start_session {channels:"cpu,gpu,frame"}` → active File trace; `insights.snapshot {}` (no `filePath`) → **`{"status":"snapshot_written","filePath":""}`** while a fresh engine-auto-named `Saved/Profiling/20260618_232626_094ED0.utrace` (34,030,745 bytes) appeared on disk at call time. Same call, immediately after, `insights.snapshot {filePath:"X:/.../Saved/Profiling/replay_explicit_snapshot.utrace"}` → `{"status":"snapshot_written","filePath":"X:/.../Saved/Profiling/replay_explicit_snapshot.utrace"}` (explicit path round-trips). Asymmetry and the self-contradictory `snapshot_written`+empty-`filePath` pair reproduce exactly as in `#1`. Source unchanged — `InsightsHandler.cpp` line 124 `Result->SetStringField(TEXT("filePath"), FilePath)` still echoes the empty input. No new ticket; confirms `#1` is live and not a flake.
- `#3-reword-and-fix` `IN-REVIEW` developer — Reworded the **Fix:** to retarget it onto the engine's `FTraceAuxiliary::OnSnapshotSaved` delegate and away from the two flawed approaches the original ticket suggested (the mtime-sort races a concurrent capture; `ResolveActiveTracePath()`/`GetTraceDestinationString()` returns the *active session* path, a different file from the snapshot — verified against engine source `TraceAuxiliary.h:117,322` + `TraceAuxiliary.cpp:937-963`, where `WriteSnapshot` broadcasts `OnSnapshotSaved(EConnectionType::File, NativePath)` synchronously with the fully-resolved auto-name). Implemented: added `PinWrightRpc::Insights::WriteSnapshotResolvingPath(RequestedFilePath, OutResolvedPath)` that binds a temporary lambda to `OnSnapshotSaved`, calls `FTraceAuxiliary::WriteSnapshot`, unbinds, and returns the captured native path (falling back to the requested path only when the delegate yields nothing). The `insights.snapshot` handler now reports that resolved path in `filePath` plus a `pathResolved` bool, so an omitted path no longer round-trips as `""`. Files: `Source/PinWright/Private/Handlers/Debug/InsightsHandler.cpp` (helper + handler + summary/param-doc), `Source/PinWright/Private/Handlers/Debug/InsightsHandlerInternal.h` (test-visible declaration). Test: added `PinWright.insights.snapshot.OmittedPathResolvesAutoName` in `Source/PinWright/Private/Tests/InsightsHandlerTest.cpp` — calls the production helper with an empty requested path and asserts the resolved path is non-empty and ends in `.utrace` (cleans up the artifact; warns-and-skips if WriteSnapshot is unavailable). Would fail if reverted to echoing the empty input.
