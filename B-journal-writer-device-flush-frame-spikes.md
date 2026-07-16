---
id: B-journal-writer-device-flush-frame-spikes
title: "Journal NDJSON writer fsyncs the physical device every frame from the game thread — 270–400 ms gameplay spikes 1–2×/sec, unplayable in every mode"
status: IN-REVIEW
severity: High
category: bug
tags: [recorder, journal, ndjson-writer, fsync, FlushFileBuffers, performance, hitch, game-thread]
encounters: 1
lastSeen: 2026-07-16T15:11:03+03:00
---

# Journal writer device-flushes every frame → periodic 270–400 ms game-thread stalls

With `bJournalEnabled` (default), PIE gameplay hitched 270–400 ms 1–2 times per second in every game mode — unplayable. The user bisected it to the journal (disabling it removed the spikes entirely). An Unreal Insights trace showed GameThread `Frame`/`Tick_Core` at 265–395 ms, `WaitForTasks` ~541 ms across 73 instances per spike, worker threads flooded with hundreds of tiny Chaos solver tasks, and `GpuWork` spanning the same windows.

## Root cause

`FRecorderLifecycle::OnDrainTick` (core-ticker, i.e. inside `SCOPED_NAMED_EVENT(Tick_Core)` — the trace's scope) → `FJournalRecorder::DrainAndFlush` → `FJournalSession::Flush` → `FNdjsonSessionWriter::Flush` → `FArchiveFileWriterGeneric::Flush` → `IFileHandle::Flush` → **`FlushFileBuffers`** (WindowsPlatformFile.cpp:936). That is a full physical-media write-cache sync, executed on the game thread **every frame in which the journal wrote at least one line** — which is every frame during flight (100 Hz drone physics telemetry). Most syncs are cheap; 1–2 per second collide with the OS cache-manager lazy-writer cadence (~1 s) plus everything else the editor writes, and those stall 270–400 ms. The worker-thread "combs" and `WaitForTasks` are fallout: the 100 Hz fixed-dt async physics falls 30–40 substeps behind during the stall and fans out catch-up tasks the next WorldTick waits on. (Capture itself was already event-driven and cheap — no widget walking, no fingerprinting; the syscall was the entire cost.)

## Fix (implemented — plugin commit 0b44cc90)

`FNdjsonSessionWriter` (`Source/PinWrightRecorder/Private/NdjsonSessionWriter.h/.cpp`) drops the `FArchive` for a raw `IFileHandle` (`OpenWrite(..., bAllowRead=true)`, identical truncate+shared-read semantics) plus an in-memory UTF-8 line buffer. `Flush()` — still once per drain tick — hands the frame's lines to the OS in one plain buffered `write()` (page cache only) and never issues a device flush; the destructor flushes the remainder and closes. NDJSON bytes and per-frame visibility to concurrent readers are unchanged. Durability change (documented in `docs/wiki-src/recorder.integration.md`): an editor crash still loses nothing (bytes are in the OS page cache each frame), but the final seconds may be lost on a full OS crash / power loss.

Regression test: `PinWright.recorder.writer.FlushWriteThroughMidSession` — drives the real `FJournalRecorder` facade, asserts a marker line is readable from the session file mid-session after one drain (catches "bytes only land on close" regressions).

## History
- `#1-user-bisect` `OPEN` reporter — User report: 270–400 ms spikes 1–2×/sec in all modes, unplayable; user bisected to the journal (off = fixed). Insights trace: Tick_Core/WaitForTasks spikes at ~0.5–1 s cadence, physics catch-up task storms. Root-caused to per-frame `FlushFileBuffers` from the drain tick (full call path above).
- `#2-buffered-write-fix` `IN-REVIEW` developer — Replaced the FArchive with raw IFileHandle + line buffer; one buffered write per drain, no device flush; NDJSON output byte-identical, live-tail semantics unchanged, durability note added to recorder.integration.md. Tests: new `PinWright.recorder.writer.FlushWriteThroughMidSession` + full `PinWright.recorder` bucket green (17/17). Live verification: 60 s PIE sumo flight with journal enabled and `stat dumphitches` → a single 72 ms hitch in the whole window (previously ~60–120 hitches of 270–400 ms). Plugin commit 0b44cc90.
