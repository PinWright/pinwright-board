---
id: E-blueprint-get-no-projection-spills
title: "blueprint.get has no projection and emits per-variable metadata twice, so a 780-variable Blueprint returns 664k chars and spills"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, blueprint-get, response-size, oversized, projection, spill]
encounters: 1
lastSeen: 2026-09-30T18:52:49+03:00
---

# `blueprint.get` can't be narrowed, and repeats each variable up to three times

`blueprint.get` takes only `path` (`Source/PinWright/Private/Handlers/Blueprint/BlueprintInfoHandler.cpp:55-58`). There is no `fields`, variable-name filter, category filter or `limit`. The response comes from `BuildBlueprintSnapshot` (`Handlers/Blueprint/BlueprintHandlerUtils.cpp:1349-1369`), which carries each member variable in three places:
- `variables[]` (:1356), where each entry already embeds its own `metadata` object (`BuildVariableJson`, :1109-1111)
- `defaults{}` (:1359), holding the CDO value
- a top-level `metadata{}` (:1360-1367), which repeats the same `CollectVariableMetadata` output a second time

The size therefore grows about three times per variable. On Ultra Dynamic Sky (about 780 variables), `blueprint.get` returned `outputTooLong` at 664,661 chars, and the caller's two ad hoc parses of the spill file both failed.

Don't point callers at `blueprint.inspect` for this. It is a superset that adds graphs, components and references, and it also has no variable filter (`Handlers/Blueprint/BlueprintInspectHandler.cpp:26-32`). The `blueprint.get` section of `docs/wiki-src/blueprint.md:214-226` gives no warning about size and no narrower route.

**Workaround:** For CDO values, pass the Blueprint asset path to `property.list`. It resolves the path to the CDO (`Handlers/Utility/UtilityPropertyHandler.cpp:184-191`) and accepts a `nameMatch` substring or an exact `propertyNames` allow-list (:2080-2081); set `includeMetadata:false` to keep it inline. For variable descriptors (category, flags), read `bpir.txt`/`properties.json` from the asset-dump cache.

**Fix:**
1. Remove the top-level `metadata{}` map, which duplicates `variables[].metadata`, or gate it behind an opt-in. This alone cuts the per-variable cost by about a third.
2. Add optional `fields` (for example `["variables","functions"]`), `variableNames`/`nameMatch` and `category` filters, applied to `variables[]` and `defaults{}` together.
3. In `blueprint.md`, say that `blueprint.get` on a very large Blueprint spills, and point few-variable reads at `property.list {nameMatch|propertyNames}`.

## Distinct from
- `E-add-variable-full-snapshot-spill` (OPEN): same `BuildBlueprintSnapshot`, but that ticket is about the mutator's echo. This one is about the reader having no projection. Fix step 1 helps both.
- `E-blueprint-list-no-projection-spills` (OPEN): per-row projection on `blueprint.list`, a different verb.
- `E-inspect-object-no-projection-spills` (WONTFIX): that one was closed because `property.list` already covers it. Here `property.list` covers only the CDO values, not the variable descriptors, and the duplicated metadata is waste in this handler.
- `E-http-response-spill` (DONE): the generic spill mechanism.

## History
- `#1-ultra-dynamic-sky-spill` `OPEN` reporter — A subagent ran `blueprint.get {"path":".../Ultra_Dynamic_Sky"}` on a Blueprint with about 780 variables and got `{"outputTooLong":true,"message":"Response exceeds display limit (664661 chars ...)"}`. Both of its follow-up parses of the spill file failed. Checked against source: the handler registers only `path` (`BlueprintInfoHandler.cpp:55-58`), and `BuildBlueprintSnapshot` emits variable metadata twice, in `variables[].metadata` and the top-level `metadata{}` (`BlueprintHandlerUtils.cpp:1109-1111`, :1360-1367), plus `defaults{}`. `blueprint.inspect` is not a narrower alternative. `property.list {nameMatch|propertyNames}` on the BP path is the inline workaround for CDO values.
