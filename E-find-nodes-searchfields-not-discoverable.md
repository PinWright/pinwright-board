---
id: E-find-nodes-searchfields-not-discoverable
title: "find_nodes searchFields rejects unknown names without listing the valid ones; shortened guesses (\"title\") are not aliased"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint-graph, find-nodes, search-fields, discoverability]
---

# find_nodes searchFields rejects unknown names without listing the valid ones

`blueprint.graph.find_nodes` accepts `searchFields` ("Optional allowlist of fields to search"). The valid names — `nodeComment`, `nodeFunction`, `nodeName`, `nodeTitle`, `nodeType`, `pinDefaultObjectPath`, `pinDefaultTextValue`, `pinDefaultValue`, `pinName`, `pinSubTypeObjectName`, `pinSubTypeObjectPath`, `pinType` (the 12 in `IsKnownSearchFieldName`, `BlueprintGraphInspectionHandler.cpp:343-353`) — are documented nowhere and appear only in a successful response's `searchFields` echo. Passing the natural guess `searchFields: ["title"]` fails with `[INVALID_ARGUMENT] searchFields provided but no known field names were supplied.` (`:971-972`) — no list of valid names, no hint that "title" means "nodeTitle".

Nuance: `NormalizeKnownSearchField` (`:355-372`) already lowercases input and matches case-insensitively, so full-name case variants (`nodetitle`, `NODETITLE`) already resolve. Only **shortened / unprefixed** forms (`title`, `name`, `type`) miss — they aren't aliased to `nodeTitle`/`nodeName`/`nodeType`.

**Workaround:** Omit `searchFields` (searches all fields), or pass a full canonical name — case doesn't matter (`nodeTitle`, `nodetitle` both work).
**Fix:** (a) enumerate the 12 valid field names in the error message; (b) alias unprefixed forms `title→nodeTitle`, `name→nodeName`, `type→nodeType` in `NormalizeKnownSearchField`; (c) document the list in the `blueprint.graph.find_nodes` wiki section (`docs/wiki-src/blueprint.graph.md`).

## History
- `#1-searchfields-list-not-discoverable` `OPEN` reporter — `find_nodes {assetPath:"/App/App/UI/LobbyAndMenu/W_DroneSelect_EditDrone", query:"Break Drone Data", matchMode:"contains", searchFields:["title"]}` → `[INVALID_ARGUMENT] searchFields provided but no known field names were supplied.` Retried without `searchFields` → worked; the response's `searchFields` echo revealed the valid names. Verified in source: validation at `BlueprintGraphInspectionHandler.cpp:952-977`, canonical list at `:343-353`, case-insensitive normalize (full names only, no shortened aliases) at `:355-372`, bare error at `:971-972`. No searchFields docs in `docs/wiki-src/blueprint.graph.md`. Not a duplicate of E-find-nodes-eventgraph-default (scope) or E-find-orphaned-nodes-misleading-verb (verb).
