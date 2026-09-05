---
id: B-insights-export-write-failure
title: "`insights.export_trace` logs CSV write failures but still returns a successful export, including when no requested file was written"
status: IN-REVIEW
severity: High
category: bug
tags: [insights, trace, csv, persistence, false-success]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Trace export turns output failures into a green response

## What happens

`WriteCsvFile` calls `SaveStringToFile`, appends the path only on success, but on
failure merely logs a warning and continues (`TraceExportCore.cpp:70-85`). The
output-directory creation result is also ignored (`:621-622`). After all requested
dumps, `RunTraceExport` unconditionally sets `bSuccess = true` and returns the
possibly empty `files` array (`:712-759`).

An unwritable `outDir`, a path collision with a file, or an I/O failure can therefore
produce an overall successful `insights.export_trace` response without one or more
requested artifacts. No per-kind failure tells the caller what is missing.

## Why it matters

The output files are the purpose of the verb. A green export receipt can be used as
profiling evidence even though nothing landed on disk. Severity is High for silent
false success on the normal output path.

## What should happen

Make directory creation and every requested write return structured results. Fail
the export if a required artifact cannot be written, include per-kind errors and
the attempted paths, and verify file presence and nonzero size before success. Use
the board's atomic output-writer pattern if existing files may be replaced.

## Workaround

Use a new writable output directory for every call, then independently verify that
every expected path in `files` exists and is nonempty.

## Related

- `F-insights-export-trace` — feature ticket does not cover failed write receipts.
- `B-insights-export-gamethread-block` — independent execution/liveness defect.

## Fix

**Verdict: TRUE.** `SaveStringToFile` and output-directory creation failures were
ignored, while `RunTraceExport` unconditionally returned `bSuccess=true` even
when no requested CSV had been published.

**Changed files:**

- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Debug\TraceExportCore.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Debug\TraceExportCore.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\ErrorCodes.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\AtomicFileWriter.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Debug\TraceAnalysisHandler.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\InsightsHandlerTest.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Debug\InsightsHandlerInternal.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\docs\wiki-src\insights.md`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\docs\wiki-src\performance-profiling.headless-insights.md`

**Fix:** each CSV uses `AtomicFileWriter::WriteUtf8` with replacement policy;
publication failures propagate as `ErrorCodes::ERR_EXPORT_FAILED`, including
the attempted path and writer OS error, and success requires every requested
artifact to be present and non-empty. A requested `frame_series` with no Game
frames fails with the same typed error and attempted path instead of producing
an empty success. The test-only stage seam injects a partial temporary write to
prove cleanup and no final-file publication. Cancellation, timeout, and export
errors remove only paths successfully published by that request; pre-existing
CSV/temp artifacts are rejected before mutation, and TraceServices session teardown
remains off-thread before completion. A process-local canonical output-directory
reservation rejects concurrent requests targeting the same artifacts.
Deadline enforcement is cooperative around plugin-controlled enumeration boundaries; an individual engine aggregation or OS write is not forcibly preempted.

**Test IDs:** `PinWright.insights.export_trace.WriteFailureIsTypedAndLeavesNoPartialFile`,
`PinWright.insights.export_trace.NoGameFramesAreTypedFailure`, and
`PinWright.insights.export_trace.TimeoutIsTypedAndLeavesNoArtifacts` (injected and
forced-empty failures through `InvokeHandlerWithCapture` and the job registry;
static-only verification in this pass).

**Deliberately unchanged:** this is per-file atomic publication, not an
all-requested-files transaction; cleanup removes only newly published paths
owned by this request when it fails, while TraceServices analysis/provider
behavior is unchanged. The `AtomicFileWriter.cpp` directory hunk is retained
only to include the OS error when destination-directory creation fails.

## History
- `#1-source-scan-write-receipt` `OPEN` reporter — Source-only scan confirmed ignored directory creation, warning-only CSV write failures, and unconditional `bSuccess=true` after the dump. No build, test, editor, MCP call, or plugin edit was performed.
- `#2-atomic-export-failure` `IN-REVIEW` developer — Replaced warning-only writes with atomic publication and typed `EXPORT_FAILED` propagation, added the behavioural no-partial-file test, and documented the failure contract. No live runtime verification was performed.
- `#3-missing-artifact-and-error-detail` `IN-REVIEW` developer — Added expected-artifact verification, typed no-Game-frames failure, the matching handler test, and preserved the directory OS-error reporting needed by the atomic writer contract. No live runtime verification was performed.
- `#4-timeout-cancel-owned-cleanup` `IN-REVIEW` developer — Preserved typed write failure behavior while adding bounded timeout/cancellation cleanup ownership and forced-slow handler coverage; no all-files transaction was introduced. No live runtime verification was performed.
- `#5-cooperative-abort-quiescence` `IN-REVIEW` developer — Added cooperative abort checks inside provider enumeration, exact published-path cleanup, pre-existing-output rejection, and a test-only worker-quiesced signal required before hook reset or directory cleanup. No live runtime verification was performed.
- `#6-output-reservation-abandonment-rollback` `IN-REVIEW` developer — Added process-local canonical output-directory reservation, abandoned-request rollback, and early test-hook reset on pre-ticket failure. No live runtime verification was performed.
