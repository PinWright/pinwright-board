---
id: E-spill-threshold-measured-post-wrap
title: "Response-size budgeting convention is wrong by ~2.35x — the 10000-char spill threshold is measured on the wrapped ToolResult, which carries the payload twice"
status: DONE
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
- `#2-gate-fixed-helper-added` `IN-REVIEW` developer — The ticket's premise is stale: `MarkOversizedToolResult` no longer measures the wrapper. It measures one condensed copy of `structuredContent` (`MeasureReaderFacingCharacters`, `Utils/HttpResponseSpill.cpp`; rpc-design.md §22), already pinned by `PinWright.infra.http_response_spill.ToolResultGateMeasuresOneCondensedCopy`. So the bare-result budget really is 10,000 condensed characters, and this ticket's "~4,250" figure is the wrong number now. It had been copied into the audio handlers, their tests and `audio.analysis.md` as the governing ceiling, and `audio.synth.describe_schema` was still packing pages with a pretty-print proxy whose comment said it matched the gate. Remedy 2 is implemented: new `HttpResponseSpill::MeasureInlineCharacters(result)`, which the gate itself now calls, so the two cannot drift. `AudioSynthSchemaHandler.cpp` `PackPages` uses it in place of its local pretty `MeasureChars`, which is deleted, and `TestPwSynthRecipe.cpp` (`PinWright.audio.synth.describe_schema.Sections`) measures with it too. Behaviour change: pages hold more kinds, so `pageCount` may drop. The audio.analysis and audio.music caps are unchanged and their 4,250 pretty test budgets are kept. Their comments, and `docs/wiki-src/audio.analysis.md`, now call that budget a deliberately stricter house limit rather than the gate. Docs: header comment in `HttpResponseSpill.h`, CLAUDE.md Wire Protocol, rpc-design.md §22, CHANGELOG. New test `PinWright.infra.http_response_spill.MeasureInlineCharactersIsTheGateExactly`: a payload measured at N stays inline at threshold N and spills at N-1, and preconditions check that the pretty and wrapped proxies both over-count. Filter: `PinWright.infra.http_response_spill` plus `PinWright.audio.synth.describe_schema.Sections`. Remedy 3 (dropping the duplicated text block) is moot because the gate no longer counts it. Unverified: the claim in `audio.music.md` and `AudioMusicHandler.cpp` that `section:"all"` spills was measured under the old gate and may no longer hold.
- `#3-review-nits-model-compile-packing` `IN-REVIEW` developer — Review follow-up. `ModelHandler_FitDiagnosticLimit` (`PinWrightGeometry/.../Model/ModelCompileHandler.cpp`) now calls `HttpResponseSpill::MeasureInlineCharacters` instead of its own condensed writer. Its "exporting the measurement is the follow-up" comment was replaced, and the orphaned `CondensedJsonPrintPolicy.h` include was dropped. `PinWright.audio.synth.describe_schema.Sections` now checks that every non-last page is packed full under the spill limit's own measurement: re-adding the next page's first kind must measure over `Threshold - 512`. A packer that measures a pretty print stops about 20% early and fails this check. If no section has a page boundary, the test emits a `no-page-boundary` skip marker.
- `#4-verified-linux` `DONE` tester — Verified on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d). run3/full, non-skipped: `PinWright.infra.http_response_spill.MeasureInlineCharactersIsTheGateExactly` (a payload measured at N stays inline at threshold N and spills at N-1; the pretty and wrapped proxies both over-count) and `.ToolResultGateMeasuresOneCondensedCopy`, plus the other http_response_spill tests. Remedy 2, the exported helper the gate itself calls, is in place. Remedy 1 is documented: `CLAUDE.md` Wire Protocol (line 424) and `docs/rpc-design.md` section 22 (lines 717-731) state that the gate measures one condensed copy. The ticket's premise of ~4,250 chars is stale, and the budget is 10,000 condensed characters. Coverage limit: `PinWright.audio.synth.describe_schema.Sections` passed, but its packed-full check emitted the `no-page-boundary` skip marker. So the describe_schema packer using the helper was not exercised past a page boundary. The `audio.music` claim that `section:"all"` spills is still unverified under the new gate (developer #2).
