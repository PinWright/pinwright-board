---
id: B-insights-export-write-failure
title: "`insights.export_trace` logs CSV write failures but still returns a successful export, including when no requested file was written"
status: OPEN
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

## History
- `#1-source-scan-write-receipt` `OPEN` reporter — Source-only scan confirmed ignored directory creation, warning-only CSV write failures, and unconditional `bSuccess=true` after the dump. No build, test, editor, MCP call, or plugin edit was performed.
