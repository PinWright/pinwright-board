---
id: E-recorder-list-sessions-limit
title: "recorder.list_sessions has no limit param — overflows inline budget on long histories"
status: OPEN
severity: Low
category: ergonomic
tags: [recorder, response-size, oversized]
encounters: 5
lastSeen: 2026-06-16T13:31:59Z
---

# recorder.list_sessions has no limit param — overflows inline budget on long histories

`recorder.list_sessions` is registered with `RPC_NO_PARAMS` (RecorderHandler.cpp:58) and
serializes every recording file under `Saved/DroneRecordings` — id, absolute path, and
modified timestamp per entry (RecorderHandler.cpp:60-77). With ~32 sessions the response
hit 10,260 chars, crossed the 10,000-char spill threshold, and came back as
`outputTooLong` + a file reference under `Saved/EditorAutomation/HttpResponses/`,
forcing the agent to parse the spilled JSON on disk just to learn the most recent
session id — the single most common use case ("analyze the last run").

Sessions are already sorted newest-first by `RecorderResolver::ListSessions()`
(RecorderResolver.cpp:64-67), so ordering is not the gap — the gap is purely that the
full list is always returned. A `limit` would keep the common case inline.

The `limit` param shipped (#2/#4) but the **default of 20 still overflows**: at ~10940
chars a default call spills to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`,
forcing an extra Read before any session id is usable. Root cause of the residual
overflow is the per-entry absolute `path` (RecorderHandler.cpp:76), which is fully
derivable — `RecorderResolver::ListSessions` builds `Path = Combine(RecordingsDir(),
File)` and `Id = GetBaseFilename(File)`, and `ResolvePath` reconstructs the path from id
alone (RecorderResolver.cpp:55-57, 27-40). So the `path` string roughly doubles each
entry's bytes for no information gain.

**Workaround:** read the spilled response file and take the first entries — the array
is already newest-first, so no client-side re-sorting is actually needed.
**Fix:** drop the derivable absolute `path` from the default response (or gate it behind a
`verbose`/`fields` arg), and/or lower the default `limit`. Verify whether dropping `path`
alone brings 20 entries under the 10000-char threshold; if not, reduce the default.

## History
- `#1-initial-repro` `OPEN` reporter — `call("recorder.list_sessions", {})` with ~32 sessions returned `outputTooLong` with payload spilled to `Saved/EditorAutomation/HttpResponses/.../...json`; agent had to run PowerShell ConvertFrom-Json to extract the latest 6 ids. Source check confirms `RPC_NO_PARAMS` and no truncation in the handler; the resolver already sorts newest-first, so only a `limit` param is missing.
- `#2-limit-param-added` `IN-REVIEW` developer — replaced RPC_NO_PARAMS with an optional 'limit' param (RPC_PARAM_DEF, default 20, 0 = all) on recorder.list_sessions in RecorderHandler.cpp; the handler now truncates after RecorderResolver's existing newest-first sort while totalCount keeps reporting the untruncated count so elision is detectable. Summary string updated to document the ordering and default (auto-generated wiki picks it up; no recorder wiki-src overlay exists). Regression test FRecorderListSessionsLimitTest added in Tests/Recorder/RecorderListSessionsLimitTest.cpp planting 3 temp session files and asserting limit:2 returns exactly 2 with totalCount detecting elision and limit:0 returns all.
- `#3-skip-no-mcp-tool` `SKIP` tester — Cannot exercise the fix: the `mcp__editor-automation__call` MCP tool is not available in this session (not bound, not in the deferred-tool list, and ToolSearch returns no match), so no live `recorder.list_sessions` call with `limit`/`totalCount` could be made. This is a behavioral fix (RPC param + truncation), not file/doc state, so per protocol source review alone is not PASS — leaving IN-REVIEW for a session with the editor MCP surface available.
- `#4-verify-fix` `DONE` tester — Verified live: `call("recorder.list_sessions", {limit:2})` returned exactly 2 sessions with `totalCount:20` (truncation works, elision detectable); `call("recorder.list_sessions", {limit:0})` returned the full untruncated list (spilled at 10,260 chars, confirming "0 = all"). The `limit` param is accepted (no INVALID_PARAMS) and behaves per the IN-REVIEW contract.
- `#5-default-still-spills` `OPEN` reporter — Reopening: the `limit` param fixed the explicit-limit case but the **default** still overflows. Two `call("recorder.list_sessions", {})` invocations this session each returned `{"outputTooLong":true,"message":"Response exceeds display limit (10940 chars, threshold 10000)","file":{...}}` and forced a follow-up Read of the spilled `HttpResponses/.../<uuid>.json` to extract ids before `describe_session`/`query` could run. Source confirms the residual cause: each of the default 20 entries carries an absolute `path` (RecorderHandler.cpp:76) that is fully derivable from `id` + the fixed Recordings dir (RecorderResolver.cpp:55-57, 27-40). Scope: trim the redundant `path` from the default response (or gate behind `verbose`/`fields`) and/or lower the default. The content/structuredContent envelope duplication that ~halves the inline budget is a generic MCP-adapter concern, tracked separately — not in scope here. Folded the duplicate proposal E-recorder-list-sessions-output-size into this entry rather than filing a near-dup.
