---
id: B-material-domain-filter-fallback
title: "material.graph.list_expression_types treats an invalid domainFilter as no filter and returns success"
status: DONE
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
- `#2-refuse-unknown-domain-echo-applied` `IN-REVIEW` developer — Reproduced in current source before fixing: `bUseDomainFilter = !DomainName.IsEmpty() && ParseMaterialDomain(...)` still fell through to the unfiltered catalog on a parse failure, and on UE < 5.6 the in-loop `#if` kept every class even for a VALID domain (the same false success). `Handlers/Material/MaterialDiscoveryHandler.cpp`: the vocabulary is one table (`MaterialDiscoveryDomainFilter::Names`, case-insensitive parse, canonical spelling); a non-empty value that does not parse returns `INVALID_ARGUMENT` (`ErrorCodes::ERR_INVALID_ARGUMENT`) naming the value and listing the six valid domains; a filtered success carries `appliedDomainFilter` (canonical name), omitted when no filter was asked for. On UE 5.3-5.5 (no `UMaterialExpression::IsAllowedIn`) a valid `domainFilter` is refused with `SendUnsupportedEngineVersion("5.6", ...)` instead of being silently ignored. **Behaviour change** for both refusals, noted in CHANGELOG. Param help, `docs/wiki-src/material.graph.md` (Discovery) and a new `docs/engine-version-support.md` row updated. Tests (`Tests/Material/TestMaterialListExpressionTypesDomainFilter.cpp`): `PinWright.material.graph.list_expression_types.DomainFilterTypoIsRefused` (`PostProces` -> not success, INVALID_ARGUMENT, message names the value and the valid list; fails on the old fallback, which returned success) and `PinWright.material.graph.list_expression_types.DomainFilterEchoesAppliedDomain` (omitted -> no echo; `postprocess` -> `appliedDomainFilter: "PostProcess"` and fewer `totalMatches` than unfiltered; on < 5.6 asserts UNSUPPORTED_ENGINE_VERSION).
- `#3-verified-linux` `DONE` tester — Verified on Linux, UE 5.8, PinWright 7230b41d (commit 664d0c68). run3/full passed non-skipped: `PinWright.material.graph.list_expression_types.DomainFilterTypoIsRefused` and `.DomainFilterEchoesAppliedDomain`, plus `.Basic` and `.LimitAndNamesOnlyProjection`. Acceptance: `PostProces` is refused `INVALID_ARGUMENT`, with the value and the six valid domains in the message. An omitted filter gives no echo. `postprocess` parses case-insensitively, echoes `appliedDomainFilter: "PostProcess"` and returns fewer `totalMatches` than the unfiltered catalog. Coverage limit, outside the ticket's acceptance: the developer also made UE 5.3-5.5 refuse a valid `domainFilter` with `UNSUPPORTED_ENGINE_VERSION` instead of ignoring it. That `#if UE_VERSION_OLDER_THAN(5,6,0)` branch is uncompiled and untested here, as `docs/engine-version-support.md` states. A tester with a 5.3-5.5 engine can confirm it.
