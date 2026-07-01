---
id: E-find-events-null-cursor
title: "recorder.find_events emits nextCursor:null when truncated — no cursor param to fetch the rest"
status: DONE
severity: Low
category: ergonomic
tags: [recorder, pagination, cursor]
---

# recorder.find_events emits nextCursor:null when truncated — no cursor param to fetch the rest

Truncation itself is correctly signaled: with 227 matching events and `limit:100`,
the response meta carried `truncated:true, totalCount:227, elided:127`. But the
envelope also renders a `nextCursor` field that is **always null** for find_events —
`FRecorderQueryEngine::FindEvents` (RecorderQueryEngine.cpp:363) never sets
`Meta.bHasNextCursor`/`Meta.NextCursor`, and the handler (RecorderHandler.cpp:203)
accepts no `cursor` param (valid params: session, name, severityMin, key, from, to,
limit). The agent sees "truncated, here's a cursor slot, cursor is null" and has no
sanctioned way to get page 2.

The sibling `recorder.get_series` with `reduction='diff'` already implements the
exact pattern: an offset cursor threaded through `RecorderReductions::Diff` and
returned as `meta.nextCursor`, consumed via a `cursor` string param.

**Workaround:** re-query with `from` set just past the last returned event's `t`.
Lossy at the boundary — events sharing that exact timestamp are either skipped
(`from` strictly greater) or re-fetched (`from` equal); dedupe on event `id`.
**Fix:** mirror the get_series diff pattern — add an optional `cursor` param to
recorder.find_events, treat it as an offset into the sorted match list (results are
already sorted by timestamp before the cap), and populate `Meta.bHasNextCursor`/
`Meta.NextCursor` when `Shown < Total`.

## History
- `#1-initial-repro` `OPEN` reporter — Session `session-20260610-143448-59B168924381076B8CA9CE9682A89F06` (227 events): `find_events {limit:100}` returned 100 events with meta `truncated:true, totalCount:227, elided:127, nextCursor:null`; agent first guessed a `tMin` param (UNKNOWN_PARAMS), then paged manually via `{from:343052.3, limit:200}`. Source check confirms FindEvents never populates the cursor and the handler takes no cursor param, while get_series diff-reduction pagination already exists in RecorderReductions::Diff.
- `#2-cursor-pagination-added` `IN-REVIEW` developer — mirrored the get_series diff-cursor pattern: FindEvents (RecorderQueryEngine.h/.cpp) now takes a cursor string parsed by the existing ParseCursor helper as an offset into the sorted match list and populates Meta.bHasNextCursor/Meta.NextCursor when matches remain; recorder.find_events (RecorderHandler.cpp) gained an optional 'cursor' param threaded into the engine. No wiki overlay edit — none exists for recorder; the param table auto-generates from RPC_PARAMS. Regression test FRecorderFindEventsPaginationTest added in Tests/Recorder/RecorderQueryEngineTests.cpp paging 5 events by 2 and asserting nextCursor values "2"/"4"/null.
- `#3-skip-no-mcp-surface` `SKIP` tester — Could not exercise the fix: the `mcp__editor-automation__call` MCP tool is not registered in this session (absent from tool list and deferred-tools list; a direct call returned "No such tool available", which is not a live transport error from a round-trip). Verifying `cursor`/`nextCursor` paging requires a live `recorder.find_events` call against a captured session, and source-only inspection of a behavioral fix is SKIP not PASS per protocol.
- `#4-verify-cursor-pagination` `DONE` tester — Verified end-to-end against the original repro session `session-20260610-143448-...` (227 events, totalCount:227). `find_events {limit:100}` → meta `truncated:true, elided:127, nextCursor:"100"` (was always null pre-fix); `{limit:100, cursor:"100"}` → `elided:27, nextCursor:"200"`; `{limit:100, cursor:"200"}` → final 27 events, `truncated:false, elided:0, nextCursor:null`. Pages tile cleanly (100+100+27=227) and the `cursor` param is in the auto-generated schema (wiki recorder.find_events.md line 18); no UNKNOWN_PARAMS.
