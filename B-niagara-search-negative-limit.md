---
id: B-niagara-search-negative-limit
title: "Niagara search verbs pass a negative limit to TArray::SetNum, which fatals the editor"
status: OPEN
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

## History
- `#1-source-scan` `OPEN` reporter -- Confirmed from current handler and UE 5.8 container source.
  No RPC was executed because this scan forbids editor runs and the failure path is fatal by source.
