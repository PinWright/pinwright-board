---
id: E-spill-threshold-measured-post-wrap
title: "Response-size budgeting convention is wrong by ~2.35x — the 10000-char spill threshold is measured on the wrapped ToolResult, which carries the payload twice"
status: OPEN
severity: Medium
category: ergonomic
tags: [response-size, spill, oversized-readback, tools-call, handler-authoring, docs, convention]
encounters: 1
lastSeen: 2026-08-16T00:00:00+03:00
---

# Handlers budget against 10000 chars, but the gate measures a ~2.35x-amplified copy

`HttpResponseSpill::MarkOversizedToolResult` is the authoritative (and only) spill
for the `tools/call` path — `Transport/McpRequestCore.cpp:865` says so explicitly.
It measures the **wrapped MCP ToolResult**, not the handler's bare result:

```cpp
// Utils/HttpResponseSpill.cpp:322-330
void MarkOversizedToolResult(const TSharedRef<FJsonObject>& ToolResult, int32 ThresholdCharacters)
{
    const int32 ClampedThreshold = ClampThresholdCharacters(ThresholdCharacters);
    const FString Body = SerializeJsonObject(ToolResult);   // <- the WRAPPED object
    if (Body.Len() <= ClampedThreshold) { return; }
```

The wrapper carries the handler's payload **twice**: JSON-escaped inside
`content[0].text`, and verbatim in `structuredContent` (built at
`HttpResponseSpill.cpp:370-371`). The escaped copy is larger than the original
because every `"` becomes `\"`, every newline `\n`, and so on.

Net effect: **a bare handler result over roughly 4,250 characters spills**, even
though the threshold constant reads `DefaultThresholdCharacters = 10000`
(`HttpResponseSpill.cpp:21`). The amplification was measured at **~2.35x** while
sizing `audio.synth.describe_schema`: a 7,704-char bare result wrapped to ~18 KB.

## Why this matters

The existing convention budgets the **bare** result against 10000 — see
`Tests/Drive/TestDriveObserveByteCap.cpp`, which is the pattern a handler author
finds when they go looking for how to size a response. That convention is a
proxy, and it under-estimates by more than 2x, so a verb carefully designed to
"fit in 10000" reliably spills to disk anyway.

Consequences for the caller: the result becomes a file reference instead of an
inline answer, which costs a round trip and defeats the sectioning/pagination
work handlers do to stay inline. Verbs that return structured JSON of any size
are affected — `describe_*`, list verbs, and any analysis verb returning a
metric report.

This is not a defect in the spill mechanism, which behaves as designed and as
documented in `CLAUDE.md`. It is that the number handler authors budget against
is not the number the gate applies.

## Suggested remedies

Any one of these would close it; the first is cheapest.

1. **Publish the effective budget.** State in `CLAUDE.md` (Wire Protocol) and in
   `rpc-design.md` that the practical inline ceiling for a bare result is
   ~4,250 chars, and that the 10000 constant applies post-wrap. One paragraph.
2. **Expose a helper** — `HttpResponseSpill::GetEffectiveBareResultBudget()` —
   so a handler that self-sections (`describe_schema` paginates against
   `GetDefaultThresholdCharacters() - 512` today) is packing against the real
   gate instead of a proxy. This is the structural fix per `rpc-design.md` §2:
   a rule nobody can forget beats a number everybody has to remember.
3. **Stop double-carrying the payload.** `content[0].text` and
   `structuredContent` are the same data. If the escaped text copy could be
   elided or truncated when `structuredContent` is present, the budget would
   roughly double for free. Needs an MCP-client compatibility check first —
   `McpRequestCore.cpp:218` notes some clients may not forward text blocks, so
   the duplication may be deliberate.

## Evidence

Mechanism verified by reading `Utils/HttpResponseSpill.cpp` and
`Transport/McpRequestCore.cpp:855-885` directly. The 2.35x ratio and the
7,704 → ~18 KB figure were measured during `audio.synth.describe_schema`
sizing, which paginates its `generators` and `effects` sections as a result.

## History

- 2026-08-16 — Filed OPEN. Found while sizing `audio.synth.describe_schema`
  during wave 1 of the audio generation subsystem; the schema work routed
  around it with self-tuning pagination, but every future structured-response
  verb hits the same proxy.
