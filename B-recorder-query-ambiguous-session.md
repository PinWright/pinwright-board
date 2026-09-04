---
id: B-recorder-query-ambiguous-session
title: "`recorder.query` treats a session argument as a filename substring and silently reads the first matching recording"
status: IN-REVIEW
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

## Fix

Root cause: the query handler had a private substring fallback that returned immediately on the first filesystem match, while the shared resolver already provided deterministic exact path, filename, and session-id resolution. The handler now calls `RecorderResolver::ResolvePath` first, then accepts a substring only when exactly one recording matches; multiple matches are sorted and returned as `AMBIGUOUS_SESSION` candidates. The regression test now drives this through the real dispatcher with two GUID-named recordings and also verifies that an exact filename proceeds to a completed query.

Files changed:
- `Plugins\PinWright\Source\PinWright\Private\Handlers\Recorder\RecorderQueryHandler.cpp`
- `Plugins\PinWright\Source\PinWright\Private\Handlers\ErrorCodes.h`
- `Plugins\PinWright\Source\PinWright\Private\Tests\Recorder\TestRecorderQuerySafety.cpp`
- `Plugins\PinWright\Docs\wiki-src\recorder.md`

Test IDs:
- `PinWright.recorder.query.ResolvesSessionsUnambiguously` — real dispatcher ambiguity response includes both candidate ids; exact filename query completes against the requested file.

Deliberately not changed: `RecorderResolver::ResolvePath` and structured recorder verbs retain their exact-only semantics. No live editor, compile, or automation run was performed.

## History
- `#1-source-scan-session-ambiguity` `OPEN` reporter — Source-only scan confirmed first-substring-match selection in the query handler and exact-only selection in the sibling shared resolver. No build, test, editor, MCP call, or plugin edit was performed.
- `#2-exact-session-resolution` `IN-REVIEW` developer — Query resolution now prefers exact shared-resolver matches and refuses ambiguous substring matches with sorted candidates; static source/registration tests were added. No compile or test run was performed.
- `#3-behavioral-session-resolution` `IN-REVIEW` developer — Replaced source-only resolver assertions with real dispatcher coverage: two GUID-named files produce `AMBIGUOUS_SESSION` with both candidates, while an exact filename reaches a completed query. No compile or test run was performed.
