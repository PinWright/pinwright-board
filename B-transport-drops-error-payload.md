---
id: B-transport-drops-error-payload
title: "MCP transport discards handler error payloads — SendError Result object never reaches the client"
status: DONE
severity: Medium
category: bug
tags: [transport, error-handling, diagnostics, recorder, python]
---

# MCP transport discards handler error payloads — SendError Result object never reaches the client

Handlers can attach a structured payload to failures via the 3-arg
`FHandlerContext::SendError(Code, Message, Result)` (`Handlers/HandlerContext.cpp:340-360`),
but the transport's completion lambda drops it: on `bSuccess == false`,
`Transport/McpTransport.cpp:497-503` builds `MakeToolCallError("[CODE] Message")` from the
code+message strings only, ignoring the `Result` parameter, and `MakeToolCallError`
(`McpTransport.cpp:144-157`) emits a text-only content block with no `structuredContent`.
Every diagnostics payload a handler carefully assembles on its error path is silently lost.

Observed symptom: `recorder.query` with a snippet doing `rec.label` attribute access on plain-dict
records raised `AttributeError`; the client saw only `[QUERY_FAILED] Python snippet raised an exception.`
The handler had in fact captured the full Python traceback — its wrapper prints
`traceback.format_exc().splitlines()` (`Handlers/Recorder/RecorderQueryHandler.cpp:98-100`) and the
handler folds it into the error payload as a `diagnostics` array (`RecorderQueryHandler.cpp:270-283`)
— but the transport threw it away. Cost two extra probe round-trips (`dir(rec)` etc.).

Other affected senders: `RecorderQueryHandler.cpp:254,266`, `Editor/EditorCommandHandler.cpp:296`
(the B-editor-save-all-pie-diagnostic enrichment never reaches MCP clients on the error path; its
DONE verification ran in-process via response capture, which does preserve `Result`),
`Blueprint/BlueprintCreationHandler.cpp:427,436`, and `Blueprint/BpirCompilerHandler.cpp`.

Related doc gap (one-line overlay fix): the `recorder.query` wiki text says "q.objects (key->record)"
without stating a record is a plain dict — say so to preempt attribute-access AttributeErrors.

**Workaround:** none from the client side; the payload is only visible in-process (test capture path).
**Fix:** in the `McpTransport.cpp` error branch, fold `Result` into the tool-call error — append the
serialized payload JSON to the text block and set `structuredContent` alongside `isError: true`.

## History
- `#1-initial-repro` `OPEN` reporter — recorder.query AttributeError on session `session-20260610-143448-59B168924381076B8CA9CE9682A89F06` returned only `[QUERY_FAILED] Python snippet raised an exception.` despite the handler embedding the full traceback as `diagnostics`; traced the loss to the transport completion lambda (`McpTransport.cpp:497-503`) ignoring `Result` on the error path. Affects all 3-arg SendError call sites, not just recorder.
- `#2-transport-error-payload-fix` `IN-REVIEW` developer — folded the 3-arg SendError Result into the tool-call error in McpTransport.cpp (MakeToolCallError now takes the payload, appends serialized JSON to the text block, sets structuredContent, and the error branch applies the same oversize-spill marking as success); fixed the second drop site found in analysis (RpcDispatcher.cpp formatter path forwarded nullptr instead of Capture.Result); amended the recorder.query summary string in RecorderQueryHandler.cpp to state records are plain dicts (the ticket's "wiki overlay" attribution was wrong — no recorder overlay file exists, the summary string is the doc source). Regression test FMcpTransportToolsCallHandlerErrorPayloadTest added in Tests/Infra/TestMcpTransport.cpp asserting structuredContent and text both carry the diagnostics payload.
- `#3-skip-mcp-tool-unavailable` `SKIP` tester — The `mcp__editor-automation__call` tool is not present in this session (not in the active tool list, and ToolSearch `select:mcp__editor-automation__call` plus keyword searches return no match), so the editor-automation MCP server is unreachable and no live round-trip is possible. The fix's surface is transport behavior (McpTransport.cpp folding the SendError Result into structuredContent + text on the error path), which can only be proven by triggering a handler error with a structured payload (e.g. recorder.query with a raising snippet) and inspecting the returned structuredContent — not by source inspection. Per protocol, a behavioral fix that cannot be exercised end-to-end via MCP is SKIP, not PASS.
- `#4-verify-fix` `DONE` tester — Reproduced the ticket's exact symptom live: `recorder.query` on session `session-20260610-143448-59B168924381076B8CA9CE9682A89F06` with snippet `return rec.label` (attribute access on a plain-dict record). The error response now carries the handler's full structured payload — `error: "runtime_error"`, a `diagnostics` array with the complete Python traceback ending in `AttributeError: 'dict' object has no attribute 'label'`, and a `meta` block — instead of the bare `[QUERY_FAILED] Python snippet raised an exception.` seen pre-fix. Confirms the transport now folds the 3-arg SendError Result into the tool-call error. Also confirmed the doc-gap fix: the `recorder.query` summary (wiki-generated/recorder.query.md, sourced from the handler summary string) now states "a record is a plain dict … attribute access raises AttributeError".
