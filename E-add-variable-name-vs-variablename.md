---
id: E-add-variable-name-vs-variablename
title: "blueprint.add_variable rejects the natural name/type params; `name` silently binds to the BP path alias, not the variable name"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, add_variable, param-alias, footgun]
encounters: 1
lastSeen: 2026-06-28T23:47:40Z
---

# `blueprint.add_variable` rejects the natural `name`/`type` params; `name` is a misleading path-alias

`blueprint.add_variable` requires `variableName` (REQ) and takes `variableType`
(OPT) (`BlueprintPropertyHandler.cpp:36-37`), but the sibling `blueprint.create`
takes `name` for the new asset, so reaching for `name`/`type` on `add_variable`
is the natural first guess. It fails — and the failure mode is a **footgun**, not
a clean rejection.

Because `blueprint.*` handlers resolve their target via `ResolveBlueprintPath`,
which accepts `name` as a wired alias for the blueprint **path**
(`requestedPath`/`path`/`name`/`blueprintPath`/`blueprint_path`; confirmed in
`E-blueprint-param-name-path-vs-assetpath` DONE, and the architecture note in
`docs/wiki-src/blueprint.md:21,26` — *"Keep `name` off explicit-path call sites
where it is a semantic object name instead of an asset-path alias"*), a call
like `add_variable {assetPath, name:"Health", type:"float"}` does **not** report
`name` as unknown. Instead `name:"Health"` silently binds to the path alias and
the genuinely-required `variableName` is reported missing:

```
[MISSING_REQUIRED_PARAM] Missing required parameter 'variableName' (type: string)
```

So on a method that also names a created entity, the generic `name` slot is
already taken by the path resolver, turning the natural guess into a confusing
"required param missing" rather than a "you meant `variableName`" hint. The error
*does* name `variableName`, so recovery is one retry — but it costs a wasted call
plus (here) a doc re-read to recover the correct `variableName`/`variableType`
names.

## What it should do

- Accept `type` as an alias for `variableType` (no collision — `type` is unused
  on this handler).
- For `name`, either de-prioritize it as a path-alias on handlers that also name
  a created entity, or have the `MISSING_REQUIRED_PARAM 'variableName'` error note
  the `name`→path collision (e.g. "did you mean `variableName`? `name` is an alias
  for the Blueprint path here"). This is the same `name`-path-alias friction class
  as `E-level-create-name-path-alias` / `E-geometry-create-name-vs-actorname`
  (create-verbs where `name` is *rejected* and should alias the entity name), but
  the **opposite direction**: here `name` is *already accepted* as a path alias, so
  it cannot simply be re-pointed — the cheap win is the `type`→`variableType` alias
  plus a collision-aware error.

## Distinct from

- `E-add-variable-type-format` (IN-REVIEW) — that ticket is the `variableType`
  *value format* (full-path / `class:`/`struct:` forms rejected); this ticket is
  the param *name* guessability (`type` vs `variableType`) and the `name`→path
  footgun. Orthogonal.
- `E-add-variable-category-param-undocumented`, `B-add-variable-default-value-ignored`,
  `E-add-variable-set-map-wrappers-undocumented` — other `add_variable` facets,
  none about the `name`/`variableName` collision or the `type` alias.

## Evidence

From the struggle audit of a clean `blueprint.compile_bpir` BPIR-idempotency task
(focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `clean`, 18 RPC
calls, transcript `agent-a6a469a604352e7e0.jsonl`). Building a fresh Actor BP
`/Game/BP_BpirIdempotencyStress` the agent first called
`blueprint.add_variable {assetPath, name:"Health", type:"float"}` →
`[MISSING_REQUIRED_PARAM] Missing required parameter 'variableName' (type: string)`;
corrected to `{variableName:"Health", variableType:"float", defaultValue:"75.0",
category:"Stats"}` which succeeded. Friction note verbatim: *"one retry on
blueprint.add_variable — I guessed name/type but the real params are
variableName/variableType (inconsistent with blueprint.create which uses name),
the error message named the right param so the fix was immediate."* Cost: one
wasted call + one doc re-read.

Severity Low: pure naming/guessability friction with immediate recovery (the
error names the correct param). `add_variable` is a common BP-authoring method,
but the per-occurrence cost is a single retry, so it stays Low.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean `blueprint.compile_bpir` idempotency task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `clean`, 18 RPC calls, zero correctness defects; transcript `agent-a6a469a604352e7e0.jsonl`). On `/Game/BP_BpirIdempotencyStress` the natural-name guess `blueprint.add_variable {assetPath, name:"Health", type:"float"}` returned `[MISSING_REQUIRED_PARAM] Missing required parameter 'variableName' (type: string)`; corrected to `{variableName, variableType, defaultValue, category}` and succeeded (1 wasted call + 1 doc re-read). Verified in source: `add_variable` declares `variableName` REQ / `variableType` OPT (`BlueprintPropertyHandler.cpp:36-37`), and `name` is a wired **path** alias via `ResolveBlueprintPath` (`E-blueprint-param-name-path-vs-assetpath` DONE; `docs/wiki-src/blueprint.md:21,26`), so `name:"Health"` silently binds to the BP path and the required `variableName` reads as missing — a footgun rather than a clean UNKNOWN_PARAMS rejection. Proposed: accept `type` as an alias for `variableType` (no collision), and either de-prioritize the `name` path-alias on entity-naming handlers or have the error note the `name`→path collision. Dedup: ripgrep across OPEN/IN-REVIEW/DONE — the `name`-path-alias family (`E-level-create-name-path-alias`, `E-geometry-create-name-vs-actorname`, `E-blueprint-param-name-path-vs-assetpath`) covers create-verbs where `name` is *rejected* (opposite direction), and the existing `add_variable` tickets (`E-add-variable-type-format` value-format, `-category-param-undocumented`, `B-add-variable-default-value-ignored`, `-set-map-wrappers-undocumented`) cover other facets — none pairs `add_variable` with the `name`/`variableName` collision + `type` alias. Genuinely new.
