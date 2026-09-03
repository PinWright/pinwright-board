---
id: B-material-domain-filter-fallback
title: "material.graph.list_expression_types treats an invalid domainFilter as no filter and returns success"
status: OPEN
severity: Medium
category: bug
tags: [material, discovery, domain-filter, validation, false-success]
---

# A typo broadens the result to every domain

The RPC advertises a closed `domainFilter` vocabulary
(`MaterialDiscoveryHandler.cpp:204-218`). It parses the value into
`bUseDomainFilter = !DomainName.IsEmpty() && ParseMaterialDomain(...)` (`:232-235`). When a caller
supplies an unknown non-empty value, parsing returns false, `bUseDomainFilter` becomes false, and
the handler skips every domain check before returning a successful unfiltered catalog.

This is the dangerous fallback direction for discovery: a typo such as `PostProces` appears to
prove that every returned expression is valid for that domain. The response does not echo an
applied-filter record that could expose the fallback.

## What it should do

Distinguish omitted from invalid. Omitted means no filter; non-empty parse failure must return
`INVALID_ARGUMENT` with the allowed domains. Echo the canonical applied domain in successful
filtered responses.

## Workaround

Use one of the exact documented names and do not infer that success means the filter parsed.

## Related

- `F-search-api-material-expressions`

## History
- `#1-source-scan` `OPEN` reporter -- Confirmed by following the parse boolean through the entire
  class loop and success response; no editor execution was needed.
