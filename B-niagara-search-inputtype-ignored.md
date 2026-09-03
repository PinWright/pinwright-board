---
id: B-niagara-search-inputtype-ignored
title: "niagara.search_modules accepts inputType but never filters on it"
status: OPEN
severity: Medium
category: bug
tags: [niagara, search-modules, input-type, accepted-and-ignored, false-success]
---

# A documented discovery filter is absent from the implementation

`niagara.search_modules` declares optional `inputType` as the DynamicInput type filter
(`NiagaraSearchHandler.cpp:402-411`). The body reads `query`, `stage`, `sourceFilter`, and `limit`
(`:421-428`) and filters usage, stage, source, and score (`:461-505`), but never reads
`inputType`. Its returned `inputs` and `outputs` arrays are explicitly empty placeholders
(`:563-565`). The verb still returns normal results and `totalMatches` success.

A caller asking for Dynamic Inputs compatible with `Niagara.Float` therefore receives the same
set as one asking for `Niagara.Vector`, with no warning that the filter was ignored. The DONE
feature ticket said signature emission was deferred, but the live RPC kept advertising the filter;
deferred implementation must not look like an applied constraint.

## What it should do

Either load/inspect DynamicInput signatures and apply the filter, or remove/refuse `inputType`
until that exists. The regression should use two types with different result sets and assert that
the filter changes `totalMatches` and every returned output type matches.

## Workaround

Omit `inputType`, inspect each candidate separately, and filter client-side.

## Related

- `F-search-api-niagara-modules`
- `E-niagara-create-node-input-type-unverifiable`

## History
- `#1-source-scan` `OPEN` reporter -- Whole-file literal scan found the declaration as the only
  `inputType` occurrence; the full handler body confirmed no indirect consumer.
