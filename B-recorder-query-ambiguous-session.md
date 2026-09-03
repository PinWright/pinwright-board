---
id: B-recorder-query-ambiguous-session
title: "`recorder.query` treats a session argument as a filename substring and silently reads the first matching recording"
status: OPEN
severity: High
category: bug
tags: [recorder, session, path-resolution, ambiguity, wrong-target]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Recorder query can silently load the wrong session

## What happens

After exact path and filename checks, `recorder.query` enumerates every `.ndjson`
recording and returns the first filename containing the caller's string
(`RecorderQueryHandler.cpp:25-61`, substring branch at `:49-58`). It neither sorts
matches nor checks that only one file matched. A value such as a date or shared ID
prefix can therefore select an arbitrary session based on filesystem enumeration
order.

The shared structured-verb resolver does not have this fallback: it accepts only an
existing path, exact filename, or exact filename with `.ndjson`
(`RecorderResolver.cpp:17-44`). The ambiguous behavior is isolated to
`recorder.query`.

## Why it matters

The query succeeds and its `meta.session` names the file only after the caller's
analysis already ran against it. Automated comparisons can silently publish facts
from a different recording. Severity is High for wrong-target data.

## What should happen

Reuse `RecorderResolver::ResolvePath`, or collect all substring matches and return
`AMBIGUOUS_SESSION` with candidate IDs unless exactly one remains. Return the
canonical resolved path/session identity before or alongside every result.

## Workaround

Pass an exact filename including `.ndjson`, or an absolute path, and compare the
returned `meta.session` before using the value.

## History
- `#1-source-scan-session-ambiguity` `OPEN` reporter — Source-only scan confirmed first-substring-match selection in the query handler and exact-only selection in the sibling shared resolver. No build, test, editor, MCP call, or plugin edit was performed.
