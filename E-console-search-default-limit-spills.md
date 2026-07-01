---
id: E-console-search-default-limit-spills
title: "system.console.search default limit (50) + fat per-row help text spills a broad cvar discovery query to file"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [console, search, cvar, response-size, oversized, default-limit, help-text, projection, system]
---

# system.console.search spills a broad cvar-discovery query to file at its default limit

`system.console.search` is the discovery surface the wiki tells agents to call
*before* `system.console_command` to find a cvar and read its `currentValue`. A
perfectly normal opening move — "what `ScreenPercentage` cvars exist and what
are they set to?" — overflows the **10000-char inline budget** and spills the
full payload to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing
a follow-up Read of the spilled file just to eyeball the discovery results. This
is the same Read-tax the no-limit-spills / default-limit-spills reader family
lands; the root cause here is two-fold:

1. **The default `limit` is 50** (`ConsoleSearchHandler.cpp:74`,
   `int32 Limit = Ctx.GetInt(TEXT("limit"), 50)`; clamped to `[1,500]` at :75).
   A broad substring like `ScreenPercentage` matches dozens of cvars
   (measured `totalMatches: 35` for `kind:"variable"` on this host — see
   Evidence), and the default keeps every one of them.

2. **Each row carries the full multi-line `GetHelp()` text** verbatim
   (`ConsoleSearchHandler.cpp:121-122`,
   `Row->SetStringField(TEXT("help"), Help ? FString(Help) : FString())`). Help
   strings are large — `xr.SecondaryScreenPercentage.HMDRenderTarget`'s help is
   ~1100 chars; `r.ScreenPercentage.MaxResolution`'s is ~450 — so the payload
   crosses the threshold at a row count well below the default 50, and even a
   manually narrowed `limit` overflows.

Both the default and an explicit small `limit=25` spilled (see Evidence), so a
caller who proactively bounds the listing *still* pays the spill-to-file + extra
Read tax on the canonical discovery call. The mitigation an agent eventually
finds — drop to `limit=1`/`2` once you know the exact name — is exactly the
exact-name-read case that `F-console-batch-get-cvar-values` covers; the broad
*discovery* call (the one this tool exists for) has no inline-friendly path.

## What it should do

Mirror the fixes already shipped/proposed for the sibling verbose readers
(`E-gameplay-tags-list-default-limit-spills`, `E-recorder-list-sessions-limit`,
`E-actor-list-no-limit-spills`, `E-skeleton-list-bones-no-limit-spills`,
`E-volume-get-info-no-limit-spills`):

- **Add a `namesOnly` / `fields` projection** so the dominant "just show me the
  matching cvar names + currentValue" case can drop the fat per-row `help`
  string (the byte dominator). A name + kind + currentValue + flags row is a
  fraction of the size and lets a much broader query stay inline; `help` is
  opt-in for the row(s) the agent then drills into.
- **Optionally lower the inline default `limit`** below 50, or have the response
  prefer a name-only projection by default and require an explicit flag to
  include `help` — keeping `totalMatches` / `truncated` (already emitted,
  `:163-164`) so elision stays detectable.
- **Docs (`docs/wiki-src/system.md`):** in the `### system.console.search`
  section note that a broad substring with the default `limit` returns full help
  text per row and can exceed the inline budget (spilling to file), and that a
  narrower `query`, a smaller `limit`, or the proposed `namesOnly`/`fields`
  projection keeps a discovery scan inline — so the overflow is a documented
  expectation rather than a surprise on the tool's own headline use.

## Distinct from

- `E-http-response-spill` (DONE) — the *generic* server-side spill mechanism
  (the file-reference fallback itself); this ticket is that a *specific verbose
  reader* (`system.console.search`) overflows by default on its headline
  discovery use, the same relationship the no-limit-spills / default-limit-spills
  family has to that mechanism.
- `E-gameplay-tags-list-default-limit-spills` (OPEN) — identical *shape*
  (default limit + fat per-row payload → spill → forced Read), different RPC.
  That one's fat column is `configFile`/`sourceType`; here it is the `help`
  string. Same proposed fix family (projection + default sizing).
- `E-console-search-stat-subcommand-blind` (OPEN) — also `system.console.search`,
  but a *false-negative* on multi-token `stat <group>` queries; this is a
  *response-size* overflow on a query that returns correctly. Orthogonal.
- `F-console-batch-get-cvar-values` (OPEN) — *step-count* on **exact-name**
  reads of already-known cvars (N searches for N names); this is the *broad
  discovery* call (one query, many matches) overflowing the inline budget.
  Different root cause, different RPC use.

## Evidence

From a realism rendering/perf-tuning task driving the gateway
(`system.inspect.*` + `system.console.*`, outcome `clean`, judge filed nothing
for the seed). The attempt's own friction note flagged it:
*"the first system.console.search for \"ScreenPercentage\" (limit 25) blew the
10000-char display threshold and was spilled to a HttpResponses .json on disk
... a default limit of 25 on a broad substring trivially exceeds the inline
cap."* Replay-confirmed verbatim against `mcp__editor-automation__call`:

- `system.console.search { query:"ScreenPercentage", kind:"variable", limit:25 }`
  → `{"outputTooLong":true,"message":"Response exceeds display limit (18107 chars, threshold 10000); full payload written to .../HttpResponses/.../<uuid>.json", ...}`
- `system.console.search { query:"ScreenPercentage", kind:"variable" }` (default
  limit 50) → `outputTooLong`, **26023 chars** > 10000, spilled to file.
- The spilled payload reports `"totalMatches": 35, "truncated": false` for
  `kind:"variable"` — 35 cvar rows, each with full multi-line `help`. Sample fat
  rows: `r.ScreenPercentage` help ≈ 400 chars
  (*"To render in lower resolution and upscale for better performance ... <=0
  compute the screen percentage is determined by r.ScreenPercentage.Default
  cvars."*); `xr.SecondaryScreenPercentage.HMDRenderTarget` help ≈ 1100 chars.
- The narrowed read-backs the agent fell back to
  (`query:"r.ScreenPercentage", limit:1`, etc.) returned inline fine — confirming
  it is the *broad-query × default-limit × fat-help* product that overflows, not
  the tool per se.

Root cause in source: `ConsoleSearchHandler.cpp:74` (default `limit` 50) +
`:121-122` (full `GetHelp()` emitted per row, no projection to drop it).

## History
- `#2-projection-fix` `IN-REVIEW` developer — Added a `namesOnly`/`fields` per-row projection to `system.console.search`, mirroring `actor.list`'s convention (`Actor/QueryHandler.cpp`), so a broad discovery scan can drop the byte-dominating multi-line `help` column and stay inline instead of spilling to file. `namesOnly:true` keeps name+kind+currentValue+flags and drops `help`; `fields=[...]` is a case-insensitive allow-list over `name`/`kind`/`help`/`currentValue`/`flags` (fields wins over namesOnly); snake_case `names_only` accepted. With neither supplied the output is byte-identical to the prior shape (`totalMatches`/`truncated` unchanged, still emitted at `:163-164` of the old file), so elision stays detectable. The optional limit-lowering sub-proposal was intentionally NOT taken — the byte dominator is per-row `help` size, not row count (`limit:25`/`20` already spilled), so projection is the fix that actually keeps a broad scan inline. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/System/ConsoleSearchHandler.cpp` (new `fields`/`namesOnly` params + conditional per-row field emission + updated summary), `docs/wiki-src/system.md` (`## Console command discovery` prose + `### system.console.search` overlay note the overflow and the `namesOnly`/`fields`/narrower-query/smaller-limit escapes). Regression test: `EditorAutomationRpcGateway.system.console.search.NamesOnlyProjectionDropsHelp` in `Source/EditorAutomationRpcGateway/Private/Tests/EditorOps/TestSystemHandlers.cpp` — registers a private probe cvar with a known help string, then routes the production handler via `InvokeHandlerWithCapture` and asserts (1) default search keeps `help`+`currentValue`, (2) `namesOnly:true` drops `help` but retains name/kind/currentValue/flags, (3) `fields:["name"]` returns only `name`; on pre-fix code the params were undeclared no-ops and `help` was always emitted, so the projection assertions fail.
- `#1-initial-repro` `OPEN` reporter — Filed from a realism rendering-tuning task (seed `system.inspect.*`/`system.console.*`, outcome clean for the seed; this is a neighbor finding on `system.console.search`). The canonical discovery call `system.console.search { query:"ScreenPercentage", kind:"variable" }` spills to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json` at its default `limit` (26023 chars > 10000 threshold); an explicit `limit:25` also spilled (18107 chars). Replay-confirmed verbatim via `mcp__editor-automation__call`. Root cause: default `limit=50` (`ConsoleSearchHandler.cpp:74`) × 35 matching `variable` rows × full multi-line `GetHelp()` per row (`:121-122`, no projection). Proposed: add a `namesOnly`/`fields` projection to drop the fat `help` column on discovery scans (and/or lower the inline default `limit` / make `help` opt-in), keeping `totalMatches`/`truncated`; document the overflow on the `### system.console.search` section of `docs/wiki-src/system.md`. Dedup: ripgrep across OPEN/DONE/WONTFIX — `E-http-response-spill` (DONE) is the generic spill mechanism (not the per-reader sizing); `E-gameplay-tags-list-default-limit-spills` (OPEN) is the same shape on a different RPC; `E-console-search-stat-subcommand-blind` (OPEN) is a multi-token false-negative on the same RPC (orthogonal, not size); `F-console-batch-get-cvar-values` (OPEN) is exact-name read step-count (not broad-discovery overflow); `F-search-api-console-commands` (DONE) added the search with no size ergonomics. No existing ticket owns the `system.console.search` response-size / default-sizing gap.
