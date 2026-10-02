---
id: E-error-catalog-generator-uncommitted
title: "docs/error-code-catalog.md is regenerated from a Python snippet pasted inside the doc, not a committed script, and the scan misses codes passed through helpers such as ResolveExpressionOrSendError"
status: WONTFIX
severity: Low
category: ergonomic
tags: [docs, error-codes, catalog, tooling, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T13:03:05Z
---

# Error-code catalog has no committed generator

`docs/error-code-catalog.md:11` says "Regenerate after any handler change with the snippet at the
bottom of this file"; the generator is a fenced Python block inside the doc (`:956-984`). No script
under `scripts/` or `Content/Python/` produces the catalog, so regenerating means copying code out of
a markdown file, nothing can run it in CI, and the snippet and the contract test it claims to mirror
(`PinWright.core.error_codes.AllEmittedCodesAreRegistered`,
`Tests/Core/TestErrorCodeRegistry.cpp`) can drift silently.

The scan is also blind to a code literal passed to any helper other than `SendError`. The doc states
this itself (`:36-39`): `EXPRESSION_NOT_FOUND`, passed as `TEXT("EXPRESSION_NOT_FOUND")` to
`ResolveExpressionOrSendError` (`Handlers/Material/MaterialAuthoringHandler.cpp:4367`), has no row,
although `ERR_EXPRESSION_NOT_FOUND` is registered in `Handlers/ErrorCodes.h:525`. How many other codes
reach the wire the same way is unknown, which is the problem with a catalog meant to be the complete
inventory.

**Impact:** docs/tooling friction; the catalog under-reports. No runtime effect.
**Fix:** move the snippet into a committed stdlib script (e.g. `Content/Python/gen_error_code_catalog.py`,
run with `uv run python -m`) and replace the doc block with its invocation. Count helper-passed codes
by matching raw `TEXT("<UPPER_SNAKE>")` literals that equal a registered `ErrorCodes::ERR_*` value
(the registry in `ErrorCodes.h` is the vocabulary), or migrate such helpers to take `ErrorCodes::ERR_*`
constants, which the scan already counts.

## History
- `#1-inline-generator-and-helper-blindness` `OPEN` reporter — Found in today's verification; verified that no generator script exists and that the doc acknowledges the `EXPRESSION_NOT_FOUND` gap. Related: `E-error-code-vocabulary-registry` (IN-REVIEW), whose migration step 1 called for the catalog to come from a script.
- `#2-stale-sweep-yagni` `WONTFIX` developer — Internal docs-tooling wish from a gap analysis, no caller encounter. The registry contract test `PinWright.core.error_codes.AllEmittedCodesAreRegistered` is the real guard; the catalog is informational and already states its helper-passed-code blind spot (`docs/error-code-catalog.md:36-39`). Regenerating from the inline snippet works. Plugin HEAD `10212ee4`.
