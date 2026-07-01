---
id: E-memory-report-no-path
title: "performance.generate_memory_report never returns the .memreport path despite its outputPath param promising it is 'Reported back in the response'"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [performance, memreport, result-misreport, docs, outputPath]
---

# `performance.generate_memory_report` writes a `.memreport` but returns no path, contradicting its own `outputPath` param doc

`performance.generate_memory_report` runs UE's `memreport` (or `memreport -full` when
`detailed:true`) and writes a breakdown to `Saved/Profiling/MemReports/<timestamp>.memreport`.
The capture works — a real, sizable detailed report lands on disk (a perf-baseline task
confirmed a 511 KB `.memreport` with a genuine texture/mesh breakdown). But the response is a
bare `{"message":"Memory report generated"}` with **no path field at all**.

What makes this more than the generic "bare message" pattern is that the method's own
`outputPath` parameter is documented to report the path back. Verbatim from the wiki
(`performance.generate_memory_report.md`, the `outputPath` param):

> `outputPath` (`string`, optional): **Reported back in the response for client convenience**;
> the actual file path is determined by UE (Saved/Profiling/MemReports/<timestamp>.memreport).

So the documented contract is explicit: the path is "Reported back in the response for client
convenience." The handler reports nothing. The caller is left to `Glob`/mtime-sort
`Saved/Profiling/MemReports/*.memreport` and guess the newest is the right one — the exact
off-MCP fallback the param doc says should be unnecessary.

This is the same class of result-misreport as the sibling `E-stop-profiling-no-uestats-path`
(the `performance.*` stat-file capture omits its `.uestats` path) and
`E-insights-snapshot-empty-filepath` (a `.utrace` reported as `filePath:""`), but it is a
**distinct method/handler** and carries a **stronger documented promise**: those rely on the
description merely mentioning the output directory, whereas here a named param explicitly
states its value is "Reported back in the response." The contradiction is therefore directly
quotable against the registered param doc, not just implied.

Root cause (`Source/.../Handlers/Debug/PerformanceHandler.cpp`,
`performance.generate_memory_report`): the handler computes
`Cmd = detailed ? "memreport -full" : "memreport"`, runs
`GEngine->Exec(GEditor->GetEditorWorldContext().World(), *Cmd)`, then
`Ctx.SendSuccess(TEXT("Memory report generated"))` — a bare success message. It never reads the
`outputPath` arg (so the "client convenience" echo is dead) and never resolves the real
UE-chosen file path, so nothing is reported back under any name.

## Repro (replay-confirmed via mcp__editor-automation__call)

- `performance.generate_memory_report {detailed:true}` -> `{"message":"Memory report generated"}` (no path field).
- `performance.generate_memory_report {detailed:true, outputPath:"X:/tmp/my_report.memreport"}` -> still `{"message":"Memory report generated"}` — the supplied `outputPath` is neither honored nor echoed, confirming the omission is consistent even when the param the doc references is provided.

## Process friction this caused (this task)

A perf-baseline task ended on "generate a detailed memory report ... and tell me where the
report file landed." The agent ran `generate_memory_report {detailed:true}`, got
`{"message":"Memory report generated"}`, and — because the param doc promised the path back —
had to leave the MCP surface, `Glob` `Saved/Profiling/MemReports/`, and check file mtime/size
to find the 511 KB report it had just produced. It then retried with `outputPath` set to
confirm the omission is consistent. Quoting the friction note: *"generate_memory_report returns
{"message":"Memory report generated"} with NO path field, even though the wiki says
outputPath/path is 'Reported back in the response for client convenience' — I had to glob the
disk and check file mtime/size to confirm the report ... actually landed."*

**Impact:** an agent cannot programmatically hand the report path to a downstream step (open
it, parse the texture/mesh hogs, attach it to a summary) from the response alone — it must
mtime-sort the directory, which is racy if any other capture is in flight. Low severity (the
file is written, not lost; a directory mtime-sort recovers it), but it is avoidable friction
that directly contradicts the method's own param documentation.

**Workaround:** after the call, `Glob`/mtime-sort `Saved/Profiling/MemReports/*.memreport` and
take the newest.

**Fix:** resolve the just-written `.memreport` (newest file under
`Saved/Profiling/MemReports/` immediately after the `memreport` exec) and return it as a result
object, e.g. `{path: "<absolute .memreport>"}`, honoring/echoing `outputPath` if the design
intends client-supplied paths — mirroring how `insights.stop_session` returns `tracePath`.
Pair with the sibling fix in `E-stop-profiling-no-uestats-path` and update the
`docs/wiki-src/performance.md` overlay so the documented "Reported back in the response" promise
matches the actual result shape.

## History
- `#1-initial-repro` `OPEN` reporter — `performance.generate_memory_report` returns `{"message":"Memory report generated"}` with **no path field**, even though its `outputPath` param doc states the value is "Reported back in the response for client convenience." Replay-confirmed twice via `mcp__editor-automation__call`: `{detailed:true}` and `{detailed:true, outputPath:"X:/tmp/my_report.memreport"}` both yield the bare message; the supplied `outputPath` is neither honored nor echoed. Source-confirmed (`PerformanceHandler.cpp`, `generate_memory_report`): runs `GEngine->Exec(... "memreport -full"/"memreport")` then `Ctx.SendSuccess("Memory report generated")` — never reads `outputPath`, never resolves the UE-chosen path. The report itself is real (perf-baseline task confirmed a 511 KB detailed `.memreport` on disk), so the call WORKS — it just omits the documented path, forcing a `Glob`/mtime-sort of `Saved/Profiling/MemReports/`. Same result-misreport class as the distinct siblings `E-stop-profiling-no-uestats-path` (`.uestats`) and `E-insights-snapshot-empty-filepath` (`.utrace`), but a separate method/handler with a stronger, directly-quotable param-doc promise. Workaround: mtime-sort the dir. Fix: return a `{path}` result; align `docs/wiki-src/performance.md`.
- `#2-additional-detailed-false` `OPEN` reporter — Additional evidence (still OPEN, current build): replay-confirmed the omission also holds for the **non-detailed** path (`detailed:false`), which the `#1` repro had only covered for `detailed:true`. Via `mcp__editor-automation__call`: `performance.generate_memory_report {detailed:false}` → `{"message":"Memory report generated"}` and `performance.generate_memory_report {detailed:false, outputPath:"X:/tmp/mytest.memreport"}` → identical `{"message":"Memory report generated"}` — no path field, supplied `outputPath` neither honored nor echoed, for both the `memreport` and `memreport -full` variants. Surfaced again by a profiling-pass task (seed `performance.start_profiling`) whose final step asked to "report back where that memreport landed"; the agent got the bare message and had to mtime/PID-sort `Saved/Profiling/MemReports/` to recover the file, exactly the off-MCP fallback the param doc says should be unnecessary. No new fix needed — same handler/root cause as `#1`.
- `#3-retriage` `OPEN` triage — Low→Medium: readback omits the .memreport path its own param doc promises, forcing a Glob/mtime-sort fallback, rare perf path.
- `#4-fix` `IN-REVIEW` developer — Fixed: the handler now resolves and returns the .memreport path, honoring the outputPath param doc's "Reported back in the response" promise. Root cause was that `memreport` DEFERS its write (UEngine::HandleMemReportCommand only queues "MemReportDeferred ..." onto GEngine->DeferredCommands, which runs a tick later after a forced full-purge GC), so "newest file immediately after exec" would resolve a stale prior report (the adversarial-lens correction). The fix issues the deferred command form DIRECTLY — `MemReportDeferred -NAME=<unique-GUID-token> [-FULL]` routes straight to HandleMemReportDeferredCommand in the SAME Exec call (deterministic synchronous write), then FindFilesRecursive locates the exact `*<token>*.memreport` under <ProfilingDir>/MemReports (race-free even under concurrent captures; tolerates UE's Pid<N>_ leaf prefix on editor/server builds). Response now carries `{message, detailed, path: <absolute>}` (plus `requestedOutputPath` echo when the caller supplies `outputPath`, and a `reportsDir`/`pathResolved:false` fallback if the file can't be located), mirroring the in-repo `insights.stop_session` tracePath pattern. Files: `Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp` (handler + corrected param/summary docs). Test: added `PinWright.performance.generate_memory_report.ReturnsResolvedPath` in `Source/PinWright/Private/Tests/EditorOps/TestDebugHandlers.cpp` — exercises the production handler and asserts the success response carries a non-empty `path` ending in `.memreport` that exists on disk (cleans up the artifact); guarded for UE 5.4 GC-crash and commandlet/no-GEditor contexts. Would fail if reverted to the bare-message SendSuccess.
