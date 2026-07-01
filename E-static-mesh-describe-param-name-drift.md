---
id: E-static-mesh-describe-param-name-drift
title: "static_mesh.describe requires 'assetPath' with no 'path' alias — the read-back asset slot drifts like the rest of the path/assetPath family"
status: OPEN
severity: Low
category: ergonomic
tags: [static-mesh, param-alias, path, assetpath, describe, drift, docs]
encounters: 2
lastSeen: 2026-06-30T00:00:00Z
---

# `static_mesh.describe` rejects `path`, requires `assetPath` — same param-name drift as the blueprint / widget / asset family

Sibling of the established path/assetPath param-alias-drift family:
`E-blueprint-param-name-path-vs-assetpath` (DONE, the canonical dispatcher
`FParamSpec` alias machinery), `E-widget-asset-path-alias-drift` (DONE),
`E-material-editor-param-name-drift` (DONE),
`E-asset-path-vs-assetpath-list-drift` (OPEN, the `asset.list`->`asset.exists`/
`asset.dump` read-back chain). The `static_mesh.*` namespace was not part of any
of those sweeps, so its read-back asset slot still drifts.

`static_mesh.describe` declares its asset slot as **`assetPath` only**:

- `StaticMeshDescribeHandler.cpp:9-13` — `RPC_PARAM_REQ("assetPath", "string",
  "Static mesh asset path")`, no alias metadata.
- `StaticMeshDescribeHandler.cpp:16` — body reads via the **single-key** overload
  `Ctx.RequireAssetPath(TEXT("assetPath"), AssetPath)`. Note there is already a
  multi-key overload `RequireAssetPath(const TArray<FString>&, ...)`
  (HandlerContext.cpp:131) that accepts several spellings — this handler does not
  use it, and even if it did, the dispatcher would still reject `path` at the
  wire level because the `RPC_PARAM_REQ` spec carries no alias (the exact
  mechanism documented in `E-blueprint-param-name-path-vs-assetpath #3`:
  `RpcDispatcher::ValidateHandlerParams` checks the declared name before the
  body runs).

So an agent that passes the natural `path` spelling for an asset gets a hard
`[MISSING_REQUIRED_PARAM] Missing required parameter 'assetPath' (type: string)`
before the handler executes. CLAUDE.md's "camelCase and snake_case aliases" rule
does not cover this — `path` and `assetPath` are distinct names, not casing
variants.

## Repro (verbatim, from the audited task)

A read-only static-mesh optimization audit (describe-only pass over 12 SM_
props) opened with:

1. `static_mesh.describe {path:"/Game/ExampleContent/Blueprints/Meshes/SM_Door"}`
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'assetPath' (type: string)`
   Retry with `{assetPath:"..."}` → full metadata returned.

All 11 subsequent `static_mesh.describe` calls used `assetPath` and succeeded.

Friction note (verbatim): "first describe call used arg key 'path' and got a
clean MISSING_REQUIRED_PARAM error pointing to 'assetPath'; corrected
immediately." One misuse-then-correct round-trip; error text is accurate (not
misleading); zero blocked progress. Pure guessability / first-call round-trip
overhead — the exact shape of the precedent family.

The drift trap is sharpened by the namespace itself: `static_mesh.describe` is
documented to return "the same JSON shape as static_mesh.json asset dumps", and
the asset-dump verb the agent would have reached for it through (`asset.dump`)
also requires `assetPath` — but the parallel discovery verb `asset.list`
*teaches* the slot name `path` (`E-asset-path-vs-assetpath-list-drift`), so an
agent primed on a list->describe chain naturally reuses `path` and gets rejected.

## What it should do

Reuse the dispatcher `FParamSpec` alias machinery landed in
`E-blueprint-param-name-path-vs-assetpath #4` (the change that made alias-only
required params validate at the wire level). Annotate the `static_mesh.describe`
`assetPath` spec with the shared path alias set (`path`, plus the snake/camel
variants already standardized for the blueprint and widget slots) and read via
the multi-key `RequireAssetPath` overload that already exists. Standardize the
alias *set*, not the canonical name, so existing `assetPath` callers keep
working. Do not solve this by making the canonical param optional per-handler
(the explicit anti-pattern called out in `E-blueprint-param-name-path-vs-assetpath
#3` — it leaves aliases undiscoverable).

Each `static_mesh.*` verb's wiki page is individually correct (it documents its
own `assetPath` slot), so the primary defect is the alias gap, not docs. The
secondary `docs` angle: the `docs/wiki-src/static_mesh.md` overlay does not
surface that the asset slot is the same `path`/`assetPath` family the agent has
seen elsewhere, so an agent that doesn't read the page first guesses `path` and
eats the round-trip. Once aliases land, a one-line note on the namespace overlay
that `path`/`assetPath` are interchangeable closes the discovery gap.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a read-only
  static-mesh optimization audit task (12 SM_ props described; outcome clean).
  First `static_mesh.describe` call used `path` → `[MISSING_REQUIRED_PARAM]
  ... 'assetPath'`, corrected to `assetPath` on retry; the other 11 describe
  calls all used `assetPath` and succeeded. Source: `StaticMeshDescribeHandler.cpp:12`
  declares `RPC_PARAM_REQ("assetPath", ...)` with no alias, body reads via the
  single-key `RequireAssetPath(TEXT("assetPath"), ...)` at :16 (the multi-key
  alias-accepting overload at HandlerContext.cpp:131 is unused, and the
  dispatcher would reject `path` at the spec layer regardless). New namespace,
  not covered by `E-blueprint-param-name-path-vs-assetpath` (DONE),
  `E-widget-asset-path-alias-drift` (DONE), `E-material-editor-param-name-drift`
  (DONE), or `E-asset-path-vs-assetpath-list-drift` (OPEN). Distinct PROCESS
  angle: the judge filed nothing (outcome clean), and no sibling ticket touches
  the `static_mesh.*` read-back slot. Fix: dispatcher `FParamSpec` alias from
  `E-blueprint-param-name-path-vs-assetpath #4`, aliasing the
  `static_mesh.describe` `assetPath` slot to accept `path`.
- `#2-revalidated-different-asset` `OPEN` reporter — Independent re-observation
  on a different asset, folded in from the rejected umbrella proposal
  `E-path-param-key-inconsistency` (a cross-namespace restatement of this whole
  drift family, declined as a duplicate of the per-namespace tickets).
  `static_mesh.describe {path:"/App/Gates/Models/Launch_Gate_7×6"}` →
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'assetPath'`; retry with
  `{assetPath:...}` → success. Same single round-trip, accurate error, zero
  blocked progress as `#1`. Source unchanged (StaticMeshDescribeHandler.cpp:12
  `RPC_PARAM_REQ("assetPath", ...)`, body :16 single-key `RequireAssetPath`),
  still no `path` alias. Same dispatcher `FParamSpec` alias fix.
