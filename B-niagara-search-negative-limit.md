---
id: B-niagara-search-negative-limit
title: "Niagara search verbs pass a negative limit to TArray::SetNum, which fatals the editor"
status: IN-REVIEW
severity: Critical
category: bug
tags: [niagara, search, limit, crash, validation]
---

# A wire integer reaches a fatal array resize

Both `niagara.graph.search_ops` and `niagara.search_modules` read `limit` with no range check
(`NiagaraSearchHandler.cpp:219` and `:428`). After collecting matches, each runs
`if (Num > Limit) SetNum(Limit)` (`:260-262` and `:523-525`). For any negative limit and any
non-empty catalog, the condition is true and `TArray::SetNum` receives that negative value.

UE 5.8's `TArray::SetNum` explicitly routes `NewNum < 0` to the `[[noreturn]]`
`UE::Core::Private::OnInvalidArrayNum`; `ContainerHelpers.cpp:6-9` implements that path as
`UE_LOGF(LogCore, Fatal, "Trying to resize TArray to an invalid size ...")`. Therefore a normal RPC
such as `niagara.graph.search_ops {limit:-1}` can terminate the shared editor.

## What it should do

Reject `limit < 0` with `INVALID_ARGUMENT` before walking the catalog. A shared bounded-limit helper
would keep the two verbs aligned. Add dispatcher tests for `-1`, `0`, and a positive cap; the
negative case must return an error without entering `SetNum`.

## Workaround

Omit `limit` or pass a non-negative value.

## Related

- `F-search-api-niagara-graph-nodes`
- `F-search-api-niagara-modules`

## Fix

Root cause: both search handlers passed the raw request `limit` to `TArray::SetNum` without rejecting negative values.

Files changed:
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraSearchHandler.h/.cpp` add the shared `NiagaraSearch::ResolveSearchLimit` helper, reject negative values with `INVALID_ARGUMENT`, clamp positive values above 500, and validate both verbs before their catalog or Asset Registry walks.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Niagara/TestNiagaraSearchLimit.cpp` add helper and handler-level coverage for negative, zero, positive, and huge limits across both search verbs, including existing number/bool/numeric-string/fractional coercions.
- `Plugins/PinWright/Docs/wiki-src/niagara.graph.md` and `Plugins/PinWright/Docs/wiki-src/niagara.md` document the shared limit contract.

Tests: added automation IDs `PinWright.niagara.search.LimitHelper` and `PinWright.niagara.search.LimitHandlers`. `LimitHelper` asserts exact coercion and bounds; `LimitHandlers` invokes both verbs for negative, zero, and huge requests (with an exact zero-row assertion and an asset-registry-dependent huge response-shape assertion). They were not run because this task is bounded to static checks.

Not changed: Niagara catalog/filtering/scoring/output schemas and unrelated Niagara `Reserve` sites; no live PIE/editor run.

## History
- `#1-source-scan` `OPEN` reporter -- Confirmed from current handler and UE 5.8 container source.
  No RPC was executed because this scan forbids editor runs and the failure path is fatal by source.
- `#2-negative-limit-guard` `IN-REVIEW` developer -- Added shared non-negative validation and a 500-row maximum for both search verbs, plus static automation coverage and contract documentation.
- `#3-limit-coercion-and-handler-coverage` `IN-REVIEW` developer -- Preserved existing integer coercions, made huge values overflow-safe before conversion, added both-verb edge coverage, and corrected the search_modules wiki heading.
