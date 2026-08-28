---
id: E-blueprint-search-path-and-autoindex
title: "`blueprint.search`: `path` is documented as a scope but is a result filter, and the expensive index is opt-out rather than opt-in"
status: IN-REVIEW
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

- `#2-redocumented-path-declined-the-autoindex-flip` `IN-REVIEW` developer — Defects 1 and 3 fixed; **defect 2's proposed default flip deliberately NOT made**, because the blocker it waited on invalidated its cost argument. `blockedBy` is stale: `B-blueprint-search-wedges-game-thread` is `DONE` and runtime-verified (26 s cold call, 1,922 editor ticks in 32.6 s, longest stall 0.247 s, no VT assert), so the "minutes of asset loading" this ticket priced the flip against no longer exists — the `autoIndex` wait is passive observation of an index the editor advances on its own tick, bounded by `timeoutSeconds` (default 100, clamped 1-600), and self-describing on expiry via `indexTimedOut` / `searchRan:false`. Flipping to `false` would trade a disclosed bounded wait for an *undisclosed wrong answer* on precisely the call where it matters most: a cold first-of-session `autoIndex:false` never kicks the index, so `IsCacheInProgress()` is false, the existing "index is still building" NOTE cannot fire, and the caller gets `Found 0 blueprints matching 'X' under /Game` in the same words a fully-indexed corpus uses for a genuine absence. Line numbers in this ticket were stale throughout (`:182`/`:185`/`:202` no longer exist; `PumpFiBIndexing` is gone entirely) — everything was re-located by symbol.
  Changed in `Handlers/Blueprint/BlueprintIndexHandler.cpp`: (1) the `path` `RPC_PARAM_DEF` description no longer reads `Content path scope for the search` — it now states that `path` is a result filter and not a cost control, that Find-in-Blueprints searches its whole index regardless, and that the sole exception is a Blueprint-free path answered before the index is touched. (2) The second half of defect 2's ask landed on the branch that actually lacked disclosure rather than as a default flip: when `autoIndex` was off and the manager still reports unindexed assets, the success `message` now appends a NOTE naming the count and telling the caller to pass `autoIndex:true`. `GetNumberUnindexedAssets()` was hoisted into a local so the field and the message cannot disagree.
  `Docs/wiki-src/blueprint.md` had already been updated for `autoIndex` / `timeoutSeconds` / `timedOut` / `searchRan` / `indexInProgress` by wave-1 commit `7826798c`, so defect 3 was partly pre-closed; what was still missing and is now added: `unindexedCount` / `unindexedAssets` as partial-corpus signals, a **field-presence** paragraph (`indexProgress`, `indexWaitSeconds`, `indexTimedOut`, `searchTimeoutSeconds`, `searchPercentComplete` are all conditional, and the empty-scope early-out returns a reduced shape omitting `searchRan`) — which also corrects the page's own claim that `searchRan` is unconditional — plus the `path`-is-not-a-cost-control framing and the `autoIndex` cost/opt-out tradeoff.
  Two contract tests added to `Tests/Blueprint/TestBlueprintSearchBoundedWait.cpp`: `PinWright.blueprint.search.PathIsDocumentedAsAResultFilter` (fails if the retired scope wording returns, or if the description stops calling `path` a filter / stops denying it is a cost control) and `PinWright.blueprint.search.AutoIndexDefaultDisclosesItsCost` (pins `autoIndex` default `true` and requires its description to name both the wait and `timeoutSeconds`; it is *meant* to fail if someone flips the default, forcing the wiki page and the message hint to move with it).
  **Known coverage gap:** the new `message` NOTE itself is not under automation test. It lives in `FinishSearchRun`, which is file-static in an anonymous namespace and reachable only with a real corpus plus a core-ticker pump; every existing `blueprint.search` test uses an empty path and takes the early-out. A test would additionally have to skip on any fully-indexed host (`unindexedAssets == 0`), emitting `PINWRIGHT_ASSERTIONS_SKIPPED` and flipping clean runs to `COMPLETED_WITH_SKIPS` for near-zero proof. Not compiled or run — orchestrator builds.
