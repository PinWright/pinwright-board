---
id: E-eqs-builtin-token-discovery
title: "eqs.* built-in token strings are undiscoverable — wiki says 'Built-in name', errors echo only the bad input, no enumeration anywhere"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [ai, eqs, env-query, authoring, error-messages, error-hint, discovery, wiki]
---

# `eqs.*` built-in token strings have no in-band discovery path

The `eqs.*` authoring methods accept short "built-in name" tokens for
generators, tests, contexts, filter kinds, and scoring equations
(resolved in `EQSHandler.cpp` via `GeneratorClassMap()`,
`TestClassMap()`, `ContextClassMap()`, `ParseFilterType()`,
`ParseScoringEquation()`). The capability works correctly — but **the
valid token strings are not enumerated anywhere an agent can reach in
band**:

1. **The wiki pages don't list them.** `eqs.add_generator` says only
   `generatorType (string, required): Built-in generator name or
   generator class path`; `eqs.add_test` says `Built-in test name or
   test class path`; `eqs.set_context_class` says `Built-in context
   name or context class path`. `eqs.set_test_filter` gives two example
   kinds (`{kind:'float_range'}` / `{kind:'bool'}`) and
   `eqs.set_test_scoring` mentions only "equation" with no values. None
   names the actual accepted tokens. The overlay source
   (`docs/wiki-src/eqs.md`) is a 2-sentence prelude with no per-method
   sections, so nothing fills the gap.

2. **The rejection errors echo only the bad input.** Every
   token-resolution failure returns a flat `[INVALID_ARGUMENT]
   Unsupported EQS <thing>: <X>` that does not list valid tokens, offer
   near matches, or point at a discovery surface.

3. **There is no sibling discovery RPC.** Unlike MetaSound — where the
   dead-end `NODE_CLASS_NOT_FOUND` error at least has a
   `search_metasound_nodes` RPC to fall back on (see
   [`E-add-metasound-node-error-no-hint`](E-add-metasound-node-error-no-hint.md))
   — the `eqs` namespace exposes no enumeration RPC. The only remaining
   source of truth is the plugin C++ itself.

Net effect: an agent that doesn't already know the exact spelling has
to read `EQSHandler.cpp` to author an EQS query. The attempt that
surfaced this self-reported exactly that — "the eqs wiki pages list
method params but do NOT enumerate the valid built-in token strings …
I had to read the plugin's EQSHandler.cpp (last-resort) to learn
'simplegrid'/'distance'/'trace'/'querier', filter 'float_range', and
'inverse_linear'." It succeeded only because it fell back to source.

## Why this is ergonomic, not a bug or gap

The calls fail *correctly* on bad input and succeed on the right
tokens; the capability (filed and shipped as
[`F-eqs-namespace-expansion`](F-eqs-namespace-expansion.md), DONE) is
fully present. The friction is purely discoverability/error-text: a
naturally-phrased token from the user's own task wording is rejected
with no route to the right one.

## Verbatim repro (replayed via `mcp__editor-automation__call`)

Against a fresh `eqs.create` query carrying a `simplegrid` generator
and one `distance` test, passing task-natural tokens:

- `eqs.add_generator { generatorType: "points_grid" }`
  (user task said "Points: Grid generator") →
  `[INVALID_ARGUMENT] Unsupported EQS generator type or class: points_grid`
  (correct token is `simplegrid`)
- `eqs.add_test { testType: "linetrace" }` →
  `[INVALID_ARGUMENT] Unsupported EQS test type or class: linetrace`
  (correct token is `trace`)
- `eqs.set_test_scoring { scoring: { equation: "inverse" } }`
  (user task said "inverse-linear scoring") →
  `[INVALID_ARGUMENT] Unsupported EQS scoring equation: inverse`
  (correct token is `inverse_linear`)
- `eqs.set_context_class { contextClass: "querier_actor" }` →
  `[INVALID_ARGUMENT] Unsupported EQS context class: querier_actor`
  (correct token is `querier`)

In every case the error string is the verbatim demonstration: it
repeats the rejected token and stops, with no list of accepted values.

## What it should do

Either (or both):
- **Enumerate the tokens in the wiki.** Add the accepted built-in names
  to the `eqs.add_generator` / `eqs.add_test` / `eqs.set_context_class`
  param descriptions and the filter-kind / scoring-equation values to
  the `set_test_filter` / `set_test_scoring` pages (e.g. via per-method
  `### ` overlay sections in `docs/wiki-src/eqs.md`). The maps in
  `EQSHandler.cpp` are the source: generators
  `actorsofclass|oncircle|simplegrid|pathinggrid|composite|donut|blueprintbase`;
  tests `distance|trace|pathfinding|pathfindingbatch|dot|gameplaytags|overlap|random|project|volume`;
  contexts `querier|item|navigationdata|blueprintbase`; filter kinds
  `minimum|maximum|range(/float_range)|match(/bool)`; equations
  `linear|inverse_linear|square|square_root|constant`.
- **Enrich the rejection errors.** Append the accepted-token list (and
  cheapest-useful near-match) to each `Unsupported EQS <thing>` error,
  e.g. `Unsupported EQS scoring equation: inverse. Valid:
  linear, inverse_linear, square, square_root, constant.` This mirrors
  the established error-hint precedent
  ([`E-add-metasound-node-error-no-hint`](E-add-metasound-node-error-no-hint.md),
  `E-make-struct-error-hint`, `E-bpir-createwidget-pin-hint`).

**Workaround:** read the four `TMap`s + two `Parse*` functions in
`Source/PinWright/Private/Handlers/AI/EQSHandler.cpp`,
or dump an existing example EQS asset and copy its tokens.

## History
- `#1-initial-repro` `OPEN` reporter — Surfaced by a realism EQS-authoring task (author `/Game/AI/EQS/FindFiringPosition`) that SUCCEEDED but only after the agent fell back to reading `EQSHandler.cpp` to learn the built-in tokens, per its own friction note. Confirmed the discovery gap on disk: the generated wiki pages (`eqs.add_generator.md` etc.) and the overlay (`docs/wiki-src/eqs.md`, a 2-sentence prelude) say only "Built-in <X> name" and never enumerate tokens. Replay-confirmed the dead-end errors via `mcp__editor-automation__call`: `points_grid` → `Unsupported EQS generator type or class: points_grid`; `linetrace` → `Unsupported EQS test type or class: linetrace`; `inverse` → `Unsupported EQS scoring equation: inverse`; `querier_actor` → `Unsupported EQS context class: querier_actor` — each echoes only the bad input, lists no valid tokens, and there is no sibling discovery RPC. Distinct from the DONE capability ticket `F-eqs-namespace-expansion` (which built the namespace/setters) — this is the error-ergonomics + wiki-enumeration angle that survives it. Same shape as `E-add-metasound-node-error-no-hint`, but worse: EQS has no `search_*` fallback at all.
- `#2-fix-enumerate-tokens` `IN-REVIEW` developer — Implemented both halves of the proposed fix. (1) Error enrichment: the five `Unsupported EQS <thing>` rejections in `Source/EditorAutomationRpcGateway/Private/Handlers/AI/EQSHandler.cpp` now append the accepted built-in token list (matching the existing "Valid: a, b, c" convention in `LevelHandler`/`MaterialAuthoringHandler`) — generator (`actorsofclass, oncircle, simplegrid, pathinggrid, composite, donut, blueprintbase`), test (`distance, trace, pathfinding, ...`), context (`querier, item, navigationdata, blueprintbase`, plus an `asset.list` pointer for project-defined contexts), filter kind (`minimum/min, maximum/max, range/float_range, match/bool`), and scoring equation (`linear, inverse_linear, square, square_root, constant`); added five small `*TokenList()` helpers next to the resolution maps. (2) Wiki enumeration: added per-method `### ` overlay sections in `docs/wiki-src/eqs.md` for `eqs.add_generator`/`add_test`/`set_context_class`/`set_test_filter`/`set_test_scoring` listing each method's accepted tokens (and calling out the natural-language traps `points_grid`→`simplegrid`, `linetrace`→`trace`, `querier_actor`→`querier`, `inverse`→`inverse_linear`); these H3 sections surface on the method pages without costing tokens on the namespace/root index. Regression test: `FEqsUnsupportedTokenErrorsEnumerateValidTokensTest` (`EditorAutomationRpcGateway.eqs.UnsupportedTokenErrorsEnumerateValidTokens`) in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestEQSHandlers.cpp` drives all five handlers with the ticket's verbatim bad tokens and asserts each rejection both echoes the bad input and lists the correct token — it fails if the hints are reverted to bare echoes. Files: `Handlers/AI/EQSHandler.cpp`, `docs/wiki-src/eqs.md`, `Tests/Gameplay/TestEQSHandlers.cpp`.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
