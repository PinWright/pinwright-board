---
id: B-rendering-lumen-software-mode-token-alias
title: "rendering.set_lumen_method rejects its own documented softwareRTMode tokens (Detail | Global)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [rendering, lumen, project-settings, enum, wiki-mismatch]
---

# rendering.set_lumen_method rejects its own documented softwareRTMode tokens (Detail | Global)

The `rendering.set_lumen_method` wiki page documents the `softwareRTMode`
parameter as accepting the friendly tokens `Detail | Global`:

- Parameters list: `softwareRTMode (string, optional): LumenSoftwareTracingMode UPROPERTY: Detail | Global`
- Notes: `softwareRTMode → LumenSoftwareTracingMode (UPROPERTY enum: Detail | Global).`

But the handler passes the supplied token straight to enum coercion against
`ELumenSoftwareTracingMode` without mapping the documented friendly token to
the raw enum identifier. Both documented tokens are rejected; only the raw
enum identifiers (`DetailTracing` / `GlobalTracing`) are accepted. This is a
valid, documented input wrongly rejected.

This is distinct from `F-rendering-project-settings` (which delivered the
`rendering.*` namespace and is DONE): that ticket's `#4` verification step
exercised `softwareRTMode:"DetailTracing"` (the raw value) and never the
documented `Detail`/`Global` tokens, so this alias gap slipped through.

## What it should do

Either (a) honor the documented friendly tokens by mapping
`Detail → DetailTracing` and `Global → GlobalTracing` before enum coercion
(matching how `reflectionMethod` already aliases `RT → Lumen` /
`SSR → ScreenSpace`), or (b) correct the wiki to document the raw enum
identifiers `DetailTracing | GlobalTracing` as the accepted tokens. Option (a)
is preferred for consistency with the `reflectionMethod` alias behavior already
documented on the same page.

## Repro (verbatim, replay-confirmed live)

RPC: `rendering.set_lumen_method`

- args `{ softwareRTMode: "Detail", save: false }` →
  `{"applied":[],"rejected":[{"name":"LumenSoftwareTracingMode","reason":"Invalid enum value 'Detail' for enum 'ELumenSoftwareTracingMode'"}],"savedTo":""}`
- args `{ softwareRTMode: "Global", save: false }` →
  `{"applied":[],"rejected":[{"name":"LumenSoftwareTracingMode","reason":"Invalid enum value 'Global' for enum 'ELumenSoftwareTracingMode'"}],"savedTo":""}`
- args `{ softwareRTMode: "DetailTracing", save: false }` (raw enum value) →
  `{"applied":["LumenSoftwareTracingMode"],"rejected":[],"savedTo":""}`

Wiki source: `Saved/EditorAutomation/wiki/rendering.set_lumen_method.md`
lines 14 and 24 both state `Detail | Global`.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live against
  mcp__editor-automation__call: both documented friendly tokens `Detail` and
  `Global` rejected with `Invalid enum value '<token>' for enum
  'ELumenSoftwareTracingMode'`; raw enum value `DetailTracing` applied cleanly.
  Wiki (`rendering.set_lumen_method`) documents `softwareRTMode: Detail | Global`
  on both the parameters list and the Notes section. Handler passes the token
  straight to enum coercion with no friendly-token alias map, unlike the
  `reflectionMethod` knob on the same page which the Notes document as aliasing
  `RT → Lumen` / `SSR → ScreenSpace`. Either honor the documented tokens via an
  alias map or correct the wiki to `DetailTracing | GlobalTracing`.
- `#2-fix-token-alias` `IN-REVIEW` developer — Implemented option (a): honor the
  documented friendly tokens. In `rendering.set_lumen_method`
  (`Source/EditorAutomationRpcGateway/Private/Handlers/Render/RenderingProjectSettingsHandler.cpp`)
  the `softwareRTMode` branch now maps `Detail → DetailTracing` and
  `Global → GlobalTracing` before enum coercion, mirroring the existing
  `reflectionMethod` alias block (`RT → Lumen` / `SSR → ScreenSpace`) a few lines
  above in the same handler. Coercion matches the enumerator identifier
  (`DetailTracing=1` / `GlobalTracing=0`), never the UMETA DisplayName, so the
  raw friendly tokens previously hit `INDEX_NONE` and were rejected. The wiki
  (`docs/wiki-src/rendering.md:49`) and param spec (handler line 191) already
  document `Detail | Global`, so no doc change was needed. Added regression test
  `FRenderingSetLumenMethodSoftwareRTModeTokenAliasTest`
  (`EditorAutomationRpcGateway.rendering.set_lumen_method.SoftwareRTModeTokenAlias`)
  in `Source/EditorAutomationRpcGateway/Private/Tests/EditorOps/TestRenderingProjectSettingsHandlers.cpp`:
  invokes the production handler via `InvokeHandlerWithCapture` with `Detail`,
  `Global`, `DetailTracing`, `GlobalTracing` (save:false), asserting each is in
  `applied[]` not `rejected[]` and lands the correct `LumenSoftwareTracingMode`
  byte on the CDO; restores the CDO in `ON_SCOPE_EXIT`. Reverting the alias makes
  the `Detail`/`Global` cases land in `rejected[]` and fail. Not compiled/tested
  here (later phase).
