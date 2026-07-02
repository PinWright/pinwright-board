---
id: E-find-nodes-no-phrase-match
title: "find_nodes tokenizes multi-word queries and matches per-term — no phrase-match mode exists or is documented"
status: OPEN
severity: Low
category: ergonomic
tags: [find_nodes, search, docs]
---

# find_nodes tokenizes multi-word queries and matches per-term — no phrase-match mode exists or is documented

`blueprint.graph.find_nodes` splits `query` on whitespace into terms
(`ParseIntoArrayWS`, `BlueprintGraphInspectionHandler.cpp:986-989`) and a node
matches when **every** term hits **some** field — AND across terms, OR across
fields (term loop, lines 1087-1104). `matchMode` is applied per *term* inside
`SearchFieldMatchesTerm` (lines 310-332), never per phrase — including `exact`,
which does `FieldValue.Equals(term)` on each token (line 322-323), not on the
whole query. So there is **no** way to match a contiguous multi-word phrase
against a field. `query:"Break Drone Data"` also matches a "Break Drone Info"
node, because "break"+"drone" hit `nodeTitle` and "data" hits some pin field.
`requireNodeFieldMatch:true` does not help — it only requires one matched field
to be node-level, which "break"/"drone" already satisfy.

The response's `termHits`/`matchedFields` expose the per-term hits, so the
behavior is inspectable after the fact — but nothing in the param text
(`"contains|prefix|exact|word (default contains)"`) or the `blueprint.graph`
wiki overlay says the query is tokenized or that `matchMode` is per-term. A
caller reaching for `exact` to phrase-match gets zero rows (no field equals a
lone "Break") and must source-dive to learn why.

**Workaround:** over-fetch with `contains`, then filter the returned `matches`
client-side on exact `nodeTitle` before acting (what this session did before
`delete_node`). A single distinctive term narrows but cannot disambiguate
"...Info" vs "...Data" node families.

**Fix:** primarily docs — add a `### blueprint.graph.find_nodes` section to
`docs/wiki-src/blueprint.graph.md` (and tighten the `matchMode`/`query` param
text) spelling out tokenization + AND-of-terms / OR-of-fields + per-term
matchMode. Optionally add a real phrase mode: treat a quoted `query` as one
term, or a `matchMode:"phrase"` that skips `ParseIntoArrayWS`.

## History
- `#1-verified-per-term-tokenization` `OPEN` reporter — Confirmed in source: `query` is whitespace-tokenized (`BlueprintGraphInspectionHandler.cpp:986-989`) and matched AND-of-terms / OR-of-fields (1087-1104); `matchMode` (incl. `exact`) applies per token via `SearchFieldMatchesTerm` (310-332, 322-323), so no phrase mode exists — the "maybe exact already phrase-matches" escape hatch does not hold. Repro: `find_nodes {assetPath:"/App/App/UI/LobbyAndMenu/W_DroneSelect_EditDrone", query:"Break Drone Data", matchMode:"contains", requireNodeFieldMatch:true}` returned 4 wanted "Break Drone Data" + 7 noise "Break Drone Info" nodes; had to filter by exact `nodeTitle` client-side. Doc gap: neither the param text nor `docs/wiki-src/blueprint.graph.md` documents the tokenization/per-term semantics.
