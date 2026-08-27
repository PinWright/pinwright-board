---
id: E-blueprint-search-path-and-autoindex
title: "`blueprint.search`: `path` is documented as a scope but is a result filter, and the expensive index is opt-out rather than opt-in"
status: OPEN
severity: Medium
category: ergonomics
tags: [blueprint-search, find-in-blueprints, docs, defaults]
blockedBy: [B-blueprint-search-wedges-game-thread]
---

# `blueprint.search` misleads callers about cost

Two ergonomic defects, both surfaced by the same live call that produced
`B-blueprint-search-wedges-game-thread`. Filed separately because they are fixable independently of the
threading rewrite, and because either alone would have prevented the incident.

## 1. `path` reads as a cost control and is not one

Declared as `RPC_PARAM_DEF("path", "string", "Content path scope for the search", "/Game")`
(`Handlers/Blueprint/BlueprintIndexHandler.cpp:182`). Any caller reads "scope" as "this is how I limit the
work". It does not limit the work. The handler's own comments say so:

```
:207-208  // Build a set of asset paths under the requested scope for post-filtering
          // FStreamSearch searches ALL blueprints — we filter results by path afterwards
:221-223  // FStreamSearch always searches the global FiB cache regardless of the
          // path filter (path is post-filter on results)
```

And the index pump at `:202` runs *before* the path query at `:217`, so even a path containing zero
Blueprints pays the full index cost. Narrowing `/Game` to a small subfolder changes nothing about the
expensive phase.

Either make it scope the work, or rename/redocument it as a result filter so the cost is not misrepresented.
Note `path` *does* skip out-of-scope Blueprints during result processing (`:375-385`), so it is not inert —
it is just not the control its description implies.

## 2. The expensive path is opt-out

`autoIndex` defaults to `true` (`:185`). So the default call is the one that can pump the entire
Find-in-Blueprints index — on this host, minutes of asset loading — and a caller who has not read the source
has no way to anticipate it.

The response already carries `unindexedCount` and `unindexedAssets`. That is enough to make the cheap path
the default and the expensive one an informed choice:

- default `autoIndex:false` — search what FiB already has, return immediately
- report e.g. *"matched 12; 8,400 assets are unindexed — pass `autoIndex:true` to include them"*

The caller then spends the time knowing they are spending it, which is the whole difference.

## 3. Docs gap

`wiki-src/blueprint.md:404-410` lists only `query`, `path`, `filter`, `limit`. It documents neither
`autoIndex` nor `timeoutSeconds`, and never mentions `timedOut` in the response — so an agent reading the
wiki cannot discover the bounded-wait behaviour added in `fe08a040` (2026-08-15), nor learn that the pump can
be turned off. The one thing the page does warn about is an unrelated tokenization quirk.

## Context for prioritisation

On a host whose `/Game` is small this verb is fine and useful — it is the only surface that answers "which
Blueprint mentions X" with no prior setup, which is why it should not be removed. The problem is purely that
nothing about the interface tells a caller when they are about to buy the expensive version.

## History

- `#1-reported-from-live-observation` `OPEN` reporter — Raised alongside `B-blueprint-search-wedges-game-thread` after a default-parameter call on a large host project. Both handler comments quoted above verified in current source at `d195a55d`; `path`-as-post-filter confirmed by reading the call ordering (pump at `:202` precedes the asset query at `:217`).
