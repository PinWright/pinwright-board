---
id: F-trace-event-scope-metadata
title: "Trace export carries no timing events, and no resolver for the scope metadata that names them"
status: OPEN
severity: Medium
category: feature
tags: [insights, trace, profiling, gpu, rdg, metadata, export, headless]
---

# Trace export carries no timing events, and no resolver for the scope metadata that names them

Diagnosing a 300 ms frame this session ended at a wall that is not a missing measurement: the
deciding datum exists in the `.utrace` PinWright already captures, and there is no surface that
returns it. Naming the cause required the Unreal Insights GUI, which an agent cannot drive.

**The datum.** GPU and RDG scopes carry *formatted* identity, not just a name. Lumen's mesh-card
capture is the case in point —
`C:/UE_5.8/Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneRendering.cpp:2868`:

```cpp
RDG_EVENT_SCOPE(GraphBuilder, "MeshCardCapture Pages:%u Draws:%u Instances:%u Tris:%u",
                NumPages, NumDraws, NumInstances, NumTris);
```

Four `uint32` counters (`:2800-2803`, accumulated under `#if RDG_EVENTS != RDG_EVENTS_NONE` at
`:2806`) that say whether a slow frame was 12 pages or 1,200. Without them the scope is a name
with a duration and no explanation.

**Why the numbers are not in the name.** `RDG_EVENT_SCOPE` takes the *deferred* breadcrumb path.
`RenderGraphEvent.h:479` expands through `RDG_SCOPE_ARGS` (`:471`) into
`RDG_BREADCRUMB_DESC_FORWARD_VALUES` (`:446-450`), which routes valid arg types to
`RHI_BREADCRUMB_DESC_FORWARD_VALUES` and only falls back to an eager `snprintf` when
`TIsValidArgs` fails (`:406-437`) — `uint32` never does. The producer emits the *spec* once
(`RHIBreadcrumbs.h:1090` `OutputEventMetadataSpec(TraceCpuSpecId, StaticName,
TFormatString::FormatString, ...)`) and the *values* per event as a CBOR blob (`:1136`
`OutputMetadata`). So the trace stores `Format` and `FieldNames`
(`TraceServices/Model/TimingProfiler.h:35-40`) beside a per-event blob, and substitution is the
consumer's job.

**Only the GUI does that job.** The printf engine is
`Insights/TimingProfiler/Tracks/ThreadTimingTrack.cpp:451-530` (`AppendMetadataToString`), called
from `AppendInfoToTimingEventDisplayName` at `:590-620`. It exists in exactly one file, and that
file is a UI track renderer.

**The engine's own exporter drops it.**
`Developer/TraceInsights/Private/Insights/TimingProfiler/ViewModels/TimingExporter.cpp` —
`FTimingExporter::ExportTimingEvents_InitColumns` (`:337-356`, columns declared `:326-333`)
registers eight columns: `ThreadId, ThreadName, TimerId, TimerName, StartTime, EndTime, Duration,
Depth`. The set is closed — `MakeExportTimingEventsColumnList` (`:360`) rejects any name not in
it, so no caller can ask for a ninth. The split is the sharp part, and it is the wrong way round:

- `TimerId` **does** resolve the metadata indirection — `:459-463` calls
  `TimerReader.GetOriginalTimerIdFromMetadata(TimerIndex)` on a negative index, so the exporter
  knows metadata events exist.
- `TimerName` **does not** — `:466-476` writes raw `Timer->Name` and stops. `GetMetadata(` and
  `FMetadataSpec` appear zero times in the whole file.

`FExportTimingEventsParams` (`TimingExporter.h:74-80`) has no `bShowFormatInTimerNames` field
either — the flag exists only on `FExportTimersParams` (`:71`) and `FExportTimerCalleesParams`
(`:157`) — and it would not help: `MakeTimerDisplayName`
(`TraceServices/Private/Model/TimingProfiler.cpp:431-454`) appends `Spec->Format` verbatim. So
the engine's CSV would carry the literal string `MeshCardCapture Pages:%u Draws:%u Instances:%u
Tris:%u`. No exporter in the engine resolves scope metadata; the two that touch the format string
are the timer-dictionary and callee-tree exports, never the per-event export.

**PinWright never reaches even that far.** `insights.export_trace` ships four kinds, parsed at
`Source/PinWright/Private/Handlers/Debug/TraceAnalysisHandler.cpp:192-199` as a closed set —
`timer_stats | frame_series | counters | timers` — onto the `FExportKinds` bools
(`Handlers/Debug/TraceExportCore.h:16-22`). Greps over the whole plugin `Source/` tree return
**zero** hits for `ExportTimingEvents`, `FTimingExporter` and `TimingExporter`, and zero for
`GetMetadata` / `MetadataSpec` against TraceServices. Two details make the gap concrete:

1. `timers.csv` drops the format string as well — header `timer_id,name,type,file,line`
   (`TraceExportCore.cpp:542`), raw `Timer->Name` at `:554`, `MetadataSpecId` never read. The
   export cannot even report *which* timers carry deferred args.
2. The plugin's only per-event walk throws the identity away at the signature.
   `TraceExportCore.cpp:387-389`:
   ```cpp
   Timeline.EnumerateEvents(0.0, Duration,
       [&](double EventStart, double EventEnd, uint32 Depth,
           const TraceServices::FTimingProfilerEvent& /*Event*/)
   ```
   The parameter is commented out and `:391` returns early on `Depth != 0`. The timeline access
   and the enumeration callback are both already in hand; `TimerIndex` is discarded one character
   from where it would be used.

**Not covered by `F-insights-export-trace` (DONE).** That ticket delivered the four kinds above
(`:43-49`) and closed the "reach TraceServices from an agent at all" problem. It neither delivers
nor excludes per-event export — `ExportTimingEvents`, `timing_events`, `TimingExporter`, `RDG`
and `MeshCardCapture` appear nowhere in it. The gap is silent, which is why it survived a DONE.

**Fix:** both halves, and the second is the load-bearing one.

1. **A `timing_events` export kind** on `insights.export_trace`, writing the eight-column
   per-event CSV with a time window and a depth or thread filter (an unfiltered per-event dump of
   a multi-second capture is millions of rows — the row cap and `resultsTruncated` discipline
   applies).
2. **A metadata resolver**, without which (1) reproduces the engine's own defect and ships a CSV
   whose most valuable column reads `%u`. For each event whose `TimerIndex` is negative-as-signed:
   `GetOriginalTimerIdFromMetadata` for the real timer, `GetMetadata(TimerIndex)` for the CBOR
   blob, `GetMetadataSpec(Timer->MetadataSpecId)` for `Format` + `FieldNames`, then substitute.
   `AppendMetadataToString` (`ThreadTimingTrack.cpp:451-530`) is the reference implementation, and
   it is UI code — PinWright needs its own. Emit **both**: a `name` column carrying the resolved
   string and a structured `metadata` column (the `FieldNames`-keyed values), so a caller can
   filter on `Pages > 1000` without re-parsing prose. Where a spec is absent, emit the raw name
   and say so per row rather than silently degrading.

A `timing_events` kind shipped without (2) should be treated as not closing this ticket.

**Severity: Medium.** Impact class is the rubric's *"hard blocker with no workaround (a stub, a
missing verb ...), so a reasonable task is impossible"* band — the deciding datum for a frame-time
regression is captured, is in the file, and cannot be read back through any headless surface; the
only resolver in the engine is a UI widget. Not `High`: nothing here is a silent lie — the four
shipped kinds are honest about what they carry, and the `%u` a naive port would emit is visibly a
format string, not plausible wrong data. Not `Low`: this is not friction, the value is
unobtainable. Reach bumps it **down** one from the band's top, not up — `insights.export_trace` is
reached when a performance regression is being chased, not in almost every session.

**Recurring class — the deciding datum exists and is unreachable through the surface that exists
to reach it.** `F-insights-export-trace` states the shape (*"the data exists on disk, the engine
ships the library to read it, but neither of the two normal 'ask the engine to read it for me'
surfaces can reach that library from an agent"*) and this is the same shape one level in: the
library is now reachable, and the call that would answer the question is still not made.
`F-performance-frame-time-statistics` (*"Nothing in the plugin reads a frame time"*) is the same
class on live quantities. `B-pcg-connect-pins-silently-replaces-edge` is its extreme, where the
deciding bit is computed by the engine, returned across the API boundary, and discarded twice.
Here it is one `GetMetadata()` call away inside a session PinWright already opens.

## History
- `#1-initial-report` `OPEN` reporter — Root-causing a 300 ms frame could be done only in the Insights GUI. The identifying datum is `RDG_EVENT_SCOPE`'s formatted counters (`LumenSceneRendering.cpp:2868`), which travel as a spec (`Format` + `FieldNames`) plus a per-event CBOR blob because `RDG_EVENT_SCOPE` takes the deferred breadcrumb path (`RenderGraphEvent.h:446-450`, `RHIBreadcrumbs.h:1090`/`:1136`); substitution happens only in `AppendMetadataToString` (`ThreadTimingTrack.cpp:451-530`), a UI track renderer. The engine's own `FTimingExporter::ExportTimingEvents_InitColumns` (`TimingExporter.cpp:337-356`) emits eight columns and resolves the metadata indirection for `TimerId` (`:459-463`) but not for `TimerName` (`:466-476`), so its CSV would carry the raw format string; `GetMetadata(` and `FMetadataSpec` do not occur in that file. PinWright never calls it at all — `insights.export_trace` parses a closed set `timer_stats|frame_series|counters|timers` (`TraceAnalysisHandler.cpp:192-199`), zero plugin-wide hits for `ExportTimingEvents`/`FTimingExporter`/`TimingExporter`, `timers.csv` omits `MetadataSpecId` (`TraceExportCore.cpp:542`, `:554`), and the only per-event walk comments out the `FTimingProfilerEvent&` parameter (`:387-389`). `F-insights-export-trace` (DONE) delivers the four kinds and neither covers nor excludes this. Asks for a `timing_events` kind **and** a metadata resolver; the kind alone reproduces the engine's defect. All engine and plugin line numbers re-derived at UE 5.8 and plugin HEAD `ef8a1f1b`.
