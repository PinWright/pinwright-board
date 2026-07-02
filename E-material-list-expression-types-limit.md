---
id: E-material-list-expression-types-limit
title: "material.graph.list_expression_types has no limit/projection lever — a single category overflows the display"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [material, material-graph, list-expression-types, response-size, oversized, discovery, projection]
---

# `material.graph.list_expression_types` overflows the inline budget on a routine category filter

`material.graph.list_expression_types` returns the full set of matching
`UMaterialExpression` classes (className, shortName, category, description,
pin layout) in one payload with no truncation. Filtering to a single
category is the intended narrowing knob, yet even that overflows: an attempt
that called `list_expression_types(category=math)` got **53 results** that
crossed the display/spill limit and had to be **written to a file and read
back out-of-band with the Read tool** before the agent could pick the right
`className` strings.

This is the sibling of the search variant created in the same ticket. When
`F-search-api-material-expressions` (DONE) shipped this discovery pair, it
put a `limit` (default 20) on `search_expression_types` but **left
`list_expression_types` un-capped** — so the list mode, which is the natural
"show me everything in this category" call, is the one that overflows.

The friction is the same class already filed for sibling list/dump methods.
The converged fix shape for this family is NOT a bare `limit`: the per-record
payload here is heavy (9 fields: className, shortName, category, description,
caption, keywords, inputPins, outputPins, isParameter), so a `limit`-with-a-
moderate-default fix would repeat the `E-recorder-list-sessions-limit` mistake,
which shipped `limit` (default 20) yet was **reopened (still OPEN, #5)** because
the heavy per-entry payload still spilled at the default. The team has instead
converged on the validated `actor.list` projection shape — a `limit` cap **plus**
a `fields`/`namesOnly` projection that default-omits the heavy fields, with a
`totalMatches` (untruncated) count and a `truncated` flag — now also adopted by
`E-get-nodes-pins-spill-no-projection` (IN-REVIEW) for `blueprint.graph.get_nodes`.
`list_expression_types` is the next instance of the same gap, and its own sibling
`search_expression_types` already emits `totalMatches`, so mirroring that
vocabulary keeps the discovery pair consistent.

**Why it matters:**

- Category discovery is the documented happy path before authoring a graph
  by hand (the attempt's exact intent: "use the correct className strings").
  Forcing an out-of-band file read on the very first discovery call adds a
  Read round-trip to every "what nodes exist in category X" lookup.
- The math category alone is already 52–53 classes; broader/unfiltered
  listing is larger still, so the inline budget is exceeded on common input,
  not an edge case.

**Workaround:** read the spilled tool-result file from disk and scan it with
filesystem tools (exactly what the attempt did). `search_expression_types`
cannot substitute — it requires a `query` and discards zero-scoring classes, so
there is no query that cleanly enumerates a whole MenuCategory.

**Fix:** adopt the validated `actor.list` projection shape on
`material.graph.list_expression_types` (the same shape `E-get-nodes-pins-spill-
no-projection` adds to `get_nodes`), applied after the existing category/domain
filtering:
- `limit` (`RPC_PARAM_DEF`, `0` = all default, keeps the default byte-identical)
  truncating the returned array;
- a `fields` allow-list and a `namesOnly` boolean (via
  `FHandlerContext::ReadFieldProjection`) that default-omit the heavy
  description/caption/keywords/inputPins/outputPins so the common "list this
  category so I can pick a className" call stays inline even at the full count;
- additive `totalMatches` (untruncated full match count) + `truncated` fields,
  matching the field names the sibling `search_expression_types` already emits.

A bare `limit` with a moderate default is explicitly **not** sufficient here:
the 9-field record means even a 20–25-cap page can exceed the 10k budget (this
ticket's #2 independently observed the already-capped search variant overflow),
which is exactly why `E-recorder-list-sessions-limit` was reopened. The
projection lever is what keeps the discovery call inline. Scope is the list
variant only; verifying/trimming `search_expression_types`'s existing cap is a
separate concern (see #2) and should be split off, not bundled here.

## Repro

1. `material.graph.list_expression_types(category: "math")`.
2. Observed: ~53-entry `expressions[]` payload crosses the display/spill
   threshold; response comes back as a file reference (or `outputTooLong`)
   and must be Read off disk to enumerate `className`s.
3. Expected: an inline `limit`-capped page (default ~25) with `totalCount`
   reporting the full match count, so the common "list this category" call
   stays inline.

## History
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of a hand-authored material.graph task (seed `material.graph`, M_EnergyShield). Friction note: "the math list overflowed the display limit and was written to a file I had to Read." Call log shows `material.graph.list_expression_types category=math (53 results, written to file)`. Distinct from the task's tool-bug filings (`B-material-get-node-details-list-mode-errors`, `B-material-break-connections-named-pin-noop`): this is an ergonomic response-size gap, same class as DONE siblings `E-recorder-list-sessions-limit` and `E-graph-connections-pagination`. `list_expression_types` was shipped by `F-search-api-material-expressions` with a `limit` on the *search* variant only; the list variant is un-capped. Proposed fix: add optional `limit` (default ~25, 0=all) + untruncated `totalCount`, mirroring the recorder fix.
- `#2-additional-evidence-overflow-recurs-plus-search-variant` `OPEN` reporter — Cross-task aggregation: independently reproduced on a different material (seed `material.graph`, hand-authored M_StylizedOcean ocean-surface graph). Call log: `material.graph.list_expression_types category=Math (53 exprs, overflow to file)` — the SAME ~53-class Math category overflows the 10k display budget and spilled to a `Saved/EditorAutomation/HttpResponses/*.json` file the agent had to Read out-of-band, exactly as `#1`. New angle worth noting for the fix scope: the SEARCH variant overflows too in practice — `material.graph.search_expression_types query=Constant (overflow to file)` also spilled, even though `#1` notes the search variant was shipped *with* a `limit` (default 20). So either the Constant match set is wide enough that 20 capped results still exceed 10k chars, or the cap isn't applied before serialization on that path; the per-result payload (className+shortName+category+description+pin layout) is heavy enough that even a 20-cap can overflow. Reinforces `#1`'s proposed `limit`+`totalCount` on `list_expression_types`, and suggests the fix should also verify `search_expression_types`'s existing cap actually keeps the serialized response inline (or trim per-result fields / add a `fields` projection) so neither discovery method spills on common queries. Friction note verbatim: "Math and Constant lists overflowed the 10k display limit and I had to Read the spilled HttpResponses JSON files."
- `#3-reword-and-projection-fix` `IN-REVIEW` developer — Reworded from a `limit`-only "mirror E-recorder-list-sessions-limit" fix (that ticket is OPEN, reopened at its #5 because a heavy per-record payload still spilled at the default — a bare-limit fix would repeat that mistake) to the validated `actor.list` projection shape, then implemented it on `material.graph.list_expression_types` (`Source/PinWright/Private/Handlers/Material/MaterialDiscoveryHandler.cpp`). New optional params: `limit` (`RPC_PARAM_DEF`, 0=all default → default response byte-identical), `fields` (case-insensitive allow-list of className/shortName/category/description/caption/keywords/inputPins/outputPins/isParameter, via `Ctx.ReadFieldProjection`), and `namesOnly` (drops the heavy description/caption/keywords/inputPins/outputPins, keeping className/shortName/category/isParameter — the inline "list this category to pick a className" shape). `BuildExpressionRecord` now takes the lowercase want-key set and skips the inputPins/outputPins pin walks when not wanted; an empty set = emit all fields, so `search_expression_types` is unchanged. Response gains additive top-level `totalMatches` (full untruncated count) + `truncated`, reusing the field names `search_expression_types` already emits. Regression test `PinWright.material.graph.list_expression_types.LimitAndNamesOnlyProjection` (`Source/PinWright/Private/Tests/Material/TestMaterialDiscoveryHandlers.cpp`) asserts: default keeps the heavy `description`/`inputPins` fields with `totalMatches==count` and `truncated==false`; `limit:3` returns exactly 3 with `totalMatches==fullCount` and `truncated==true`; `namesOnly` keeps className/shortName/category but drops description/caption/keywords/inputPins/outputPins; an explicit `fields:[className,category]` returns only those keys — each fails on the pre-fix handler (levers silently ignored, no totalMatches/truncated). Scope kept to the list variant; the search-cap-still-overflows concern from #2 is left to a separate ticket. Not compiled/tested here (later phase).
