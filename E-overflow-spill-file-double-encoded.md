---
id: E-overflow-spill-file-double-encoded
title: "tools/call overflow spill file stores the whole MCP envelope — payload duplicated in content[0].text AND structuredContent, ~2x on disk"
status: OPEN
severity: Low
category: ergonomic
tags: [http, response-size, spill, mcp-adapter, encoding]
---

# tools/call overflow spill stores the MCP envelope, not the raw payload

When a `tools/call` result crosses the 10k spill threshold, `MarkOversizedToolResult` (`Source/PinWright/Private/Utils/HttpResponseSpill.cpp:320-370`) serializes the already-wrapped MCP `ToolResult` and writes *that* to `Saved/PinWright/HttpResponses/<ts>/<ts>_<guid>.json`. The wrapper (`MakeToolCallSuccess`, `Source/PinWright/Private/Transport/McpTransport.cpp:53-71`) already carries the payload twice — once JSON-escaped in `content[0].text`, once as real JSON under `structuredContent` — so the file is `{"content":[{"type":"text","text":"<escaped payload>"}],"structuredContent":{<same payload>},"isError":false}`: ~2x the payload on disk, and a reader must know to take `structuredContent` (one parse) or double-parse `content[0].text`.

The code confirms the duplication: `TestGetComponentsLargePayload.cpp:424` notes "the result is double-counted across content[0].text and structuredContent." No test asserts on the spill file *body* (only that it exists), so the on-disk format is unconstrained.

Distinct from `E-http-response-spill` (DONE) — that is the `/rpc` `MaybeBuildSpilledResponse` path, which writes a single-copy `{result:{...}}` and is bypassed for MCP callers — and from the `*-no-limit-spills` / `E-get-nodes-pins-spill-no-projection` family, which shrink specific method payloads. This ticket is only the on-disk encoding of the MCP-path spill file.

**Workaround:** read the file's `structuredContent` field for the raw payload (single parse); ignore `content[0].text`.
**Fix:** in `MarkOversizedToolResult`, write the payload itself (the `structuredContent` object; fall back to `content[0].text` / the full ToolResult when absent, e.g. error results) instead of the whole envelope — halves file size, hands consumers raw JSON. No in-repo tooling parses these MCP spill files (the JS adapter only touches the bypassed `/rpc` path), so the reshape is low-risk; if a consumer is later found to depend on the envelope shape, add an additive `payloadPath` sidecar instead.

## History
- `#1-initial-repro` `OPEN` reporter — Verified in source: `MarkOversizedToolResult` (`Utils/HttpResponseSpill.cpp:326`) writes `SerializeJsonObject(ToolResult)`, and `ToolResult` from `MakeToolCallSuccess` (`Transport/McpTransport.cpp:57-68`) holds the payload as both escaped `content[0].text` and `structuredContent`, so the spill file is the double-encoded MCP envelope at ~2x payload size. Session evidence: `blueprint.decompile` (18042 chars), `widget.describe` (103638), `blueprint.graph.find_nodes` (26064) all spilled with this shape this session; each read needed an unwrap step. Dedup: ripgrep across the board — no ticket covers the spill *file encoding*; `E-http-response-spill` (DONE) is the single-copy `/rpc` envelope path, the `*-no-limit-spills` family is per-method payload size. Proposed: spill only the raw payload (structuredContent), or add a `payloadPath` sidecar if the envelope shape is depended on.
