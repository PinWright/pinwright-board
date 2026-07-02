---
id: E-find-nodes-multiword-or-match
title: "`blueprint.graph.find_nodes` matchMode:contains tokenizes a multi-word query and OR-matches every sibling, then the full-pin result spills to disk"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, graph, find-nodes, search, match-mode, response-size, docs]
encounters: 1
lastSeen: 2026-07-01T10:12:18+03:00
---

# `blueprint.graph.find_nodes` multi-word query over-matches, then the over-broad result spills

Locating one specific node to use as a `blueprint.insert_bpir_at_node` splice
anchor is the textbook `find_nodes` use case, and it is awkward on two
compounding fronts, both observed in a single call.

**(1) `matchMode:"contains"` does not honor the query as one substring — it
tokenizes on whitespace and OR-matches.** A call with
`{query:"Begin B", graphName:"EventGraph", matchMode:"contains"}` split the
query into the tokens `begin` and `b` and matched **any** node whose
name/title/pin-metadata contained **either** token. So the result's *first*
match was `nodeTitle:"Event ActorBeginOverlap"` (nodeType `K2Node_Event`) — a
node that does **not** contain the literal substring `"Begin B"` at all (it
matched only on `begin` inside `ActorBeginOverlap`) — and all **6** EventGraph
nodes came back. That is the opposite of what a mode literally named `contains`
implies: a caller reasonably expects `matchMode:"contains"` to find nodes
containing the whole `"Begin B"` string and return a single unambiguous hit.
Because it could not, the caller could not use `find_nodes` to pinpoint the
splice anchor at all — it fell back to reasoning about node **x-coordinates**
(picking the `x=520` PrintString by creation order) to disambiguate the real
"Begin B" node from its five siblings.

**(2) The over-broad match then overflowed the display budget and spilled.**
Because all 6 nodes matched and `find_nodes` returns full pin metadata per node
(`pinDefaultObjectPath`, `pinDefaultTextValue`, `pinDefaultValue`, `pinName`,
`pinSubTypeObjectName/Path`, `pinType`) — `includePinDefaults` defaults on — the
response hit `{"outputTooLong":true, "message":"Response exceeds display limit
(11965 chars, threshold 10000); full payload written to
…/HttpResponses/…json"}` and spilled to disk, forcing a separate `Read` of the
spill file for what is conceptually a one-node anchor lookup. This is the same
projection-lever gap already tracked for the sibling enumerator in
`E-get-nodes-pins-spill-no-projection` (IN-REVIEW) — but on a different method
(`find_nodes`, the query-scoped search) that ticket does not name, and here it
is *compounded* by finding (1): the token over-match is what produced 6 nodes
worth of pin metadata and pushed the payload over the 10k threshold. A single
clean hit would have stayed inline.

## What it should do

- **Match semantics (primary):** make `matchMode:"contains"` treat the query as
  one literal substring (or add an explicit `exact`/`literal` mode), so a caller
  locating a specific node gets a single unambiguous hit instead of every
  sibling that shares any token. The relevant machinery is
  `EMcpSearchMatchMode` / `SearchFieldMatchesTerm` in
  `BlueprintGraphInspectionHandler.cpp` (cf. `E-blueprintgraph-handler-split`,
  lines ~198–334). At minimum, if the tokenized-OR behavior is intended,
  **document it** on `docs/wiki-src/blueprint.graph.md` (find_nodes section) so
  callers know to query by a single unique token — or by `nodeId` — rather than
  a multi-word phrase.
- **Projection (secondary, shared with `E-get-nodes-pins-spill-no-projection`):**
  give `find_nodes` a compact anchor-lookup shape — a `namesOnly`/`idsOnly`
  mode, a `fields` allow-list, or default `includePinDefaults:false` — that
  returns just `nodeId`/`nodeTitle`/`x`/`y`/`graphName` so the common
  "find the anchor to `insert_bpir_at_node` after" call stays inline and needs
  no spill Read.

## Distinct from

- `E-find-nodes-eventgraph-default` (DONE) — same method, but that ticket is the
  `graphName`-omitted EventGraph-only scoping default; this is the *query-match
  semantics* + the full-pin spill. Orthogonal.
- `E-get-nodes-pins-spill-no-projection` (IN-REVIEW) — same projection-lever
  gap and same spill family, but on `blueprint.graph.get_nodes` (enumerate-all),
  not `find_nodes` (query-scoped). Finding (2) here is the `find_nodes`
  instance of that pattern, compounded by the token over-match unique to this
  ticket.
- `E-console-search-stat-subcommand-blind` / `E-niagara-standard-stack-recipe-undocumented`
  — multi-word-query tolerance friction, but on `console.search` /
  niagara `search_modules`, different methods.

## Evidence

From the struggle audit of a clean adversarial BPIR concurrency-build task
(focus `blueprint.compile_bpir`, namespace `blueprint`, outcome **clean** — 11
calls after 14 batched wiki-nav reads, every editor mutation succeeded first-try
with zero errors, and the hunted concurrency corruption never reproduced). The
one broken assumption was the splice-anchor lookup: the single
`blueprint.graph.find_nodes {query:"Begin B", graphName:"EventGraph",
matchMode:"contains"}` call returned all 6 EventGraph nodes (first match the
non-containing `Event ActorBeginOverlap`) **and** `outputTooLong` at 11965 chars,
so the caller had to `Read` the spill file and then disambiguate the true anchor
by x-position (`x=520`). Friction note verbatim: *"find_nodes tokenized 'Begin
B' into 'begin'+'b' and matched all 6 event-graph nodes instead of just the
target, and the response exceeded the display limit and spilled to a file I had
to Read; I disambiguated the middle node by x-position/creation-order (x=520)."*
Self-labeled "Minor"; recovered without retries.

severity rationale: impact=pure friction (surprising naming/semantics + a
response-spill that only forces a Read) × reach=support method on a specific
multi-word-query shape, avoidable by querying a unique token or nodeId
(not every-session) -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean adversarial `blueprint.compile_bpir` concurrency-build task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `clean`, 11 calls all `ok`/non-error, zero retries). Two compounding CallAnalyzer findings on the support method `blueprint.graph.find_nodes`, merged (same method, same single call, causally linked): (1) `matchMode:"contains"` with query `"Begin B"` tokenized on whitespace and OR-matched all 6 EventGraph nodes — first match `Event ActorBeginOverlap`, which does not contain the literal `"Begin B"` — so the anchor could not be pinpointed and the caller fell back to x-position (`x=520`) disambiguation; (2) the over-broad 6-node result carried full per-node pin metadata (`includePinDefaults` defaults on) and overflowed the 10000-char display budget at 11965 chars, spilling to `HttpResponses/…json` and forcing an extra `Read`. Proposed: make `matchMode:"contains"` honor the whole query as one substring (or add an `exact` mode / document the tokenized-OR behavior on `docs/wiki-src/blueprint.graph.md`), and add a compact `namesOnly`/`fields`/`includePinDefaults:false` anchor-lookup projection so the lookup stays inline (shared with `E-get-nodes-pins-spill-no-projection`, a `get_nodes` instance of the same projection gap). Dedup: ripgrep across OPEN/IN-REVIEW/DONE — no ticket pairs `find_nodes` with query-tokenization or a find_nodes spill; `E-find-nodes-eventgraph-default` (DONE, graphName default) and `E-get-nodes-pins-spill-no-projection` (IN-REVIEW, different method) are distinct.
