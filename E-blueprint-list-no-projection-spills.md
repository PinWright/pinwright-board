---
id: E-blueprint-list-no-projection-spills
title: "blueprint.list has no namesOnly/fields projection, so the canonical enumerate-to-pick call overflows the inline budget and spills to disk"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, blueprint-list, response-size, oversized, projection, discovery]
encounters: 2
lastSeen: 2026-06-28T19:14:28Z
---

# `blueprint.list` has no concise/projection mode, so a routine "list BPs to pick one" overflows and spills

`blueprint.list` is the canonical discovery call — "enumerate the project's
Blueprints so I can pick one to work on." But the handler emits, **for every
row**, five string fields: `name`, `path`, `class`, `packagePath`, and (when
present) `parentClass`
(`Source/PinWright/Private/PinWright_BlueprintHandlers_List.cpp:248-258`). It has
useful *filters* (`path`, `class`, `tag`, `nameFilter`, `pathStartsWith`,
`recursive`) and *pagination* (`offset`, `limit`, default `50`) (`:20-27`), but
**no `namesOnly`/`fields` projection** to trim each row down to just the
identity (`name`+`path`) the pick step actually needs.

The practical effect: a plain enumerate (`limit:40`) returns **27334 chars**,
crosses the 10000-char inline budget, returns `outputTooLong`, and the full
payload is written to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`
(`E-http-response-spill`, DONE) — forcing the caller to `Read` the spilled file
just to scan for a Blueprint and pick one. Even the default `limit:50` overflows.
The spill mechanism worked as designed; the friction is that the verb whose job
is "list the Blueprints" has no way to stay inline for the lightweight pick shape.

## What it should do

Give `blueprint.list` the same narrowing lever its sibling read paths are
getting: a `namesOnly` boolean (or a `fields` projection) that returns just
`name`+`path` per row and omits `class`/`packagePath`/`parentClass` unless
asked. That keeps the textbook "list to pick" call inline; the verbose
full-fidelity row stays available opt-in. (The filters partly mitigate — a tight
`nameFilter`/`pathStartsWith` avoids the spill — but a caller who doesn't yet
know which BP they want can't pre-filter, which is exactly the enumerate case.)

## Distinct from

- `E-get-nodes-pins-spill-no-projection` (OPEN) — same *class* of gap (a verbose
  reader with no `namesOnly`/`fields` projection → spill), but on
  `blueprint.graph.get_nodes` (per-node pins/adjacency); this ticket is the
  asset-level `blueprint.list` discovery call. Same proposed fix shape, different
  method.
- `B-blueprint-list-class-ensure` (DONE) — same method, but that was a short-class
  ensure crash; this is response size, orthogonal.
- The `*-list-no-limit-spills` family (`E-actor-list-no-limit-spills`,
  `E-inspect-list-objects-no-limit-spills`, `E-skeleton-list-bones-no-limit-spills`,
  `E-volume-get-info-no-limit-spills`, etc.) — those readers lack a `limit`
  entirely; `blueprint.list` **has** `limit`+`offset` already, so the missing
  piece here is specifically the **per-row projection**, not pagination.
- `E-asset-list-pagination-undocumented-spills` (OPEN) — `asset.list`, lever
  exists → docs-only; here the projection lever does not exist (a code fix).

**Docs page:** `docs/wiki-src/blueprint.md` (the list method is not currently
documented there — only `blueprint.list_struct_fields` is — so the projection
option should be added alongside a `blueprint.list` section).

## Evidence

From the struggle audit of a clean, efficient BPIR positioned-custom-event
round-trip task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome
`tool_bug` filed as `F-bpir-multi-self-targets`; 7 RPCs, all `ok`/non-error,
zero retries). The very first discovery call `blueprint.list {limit:40}`
returned `outputTooLong`: *"Response exceeds display limit (27334 chars,
threshold 10000); full payload written to …20260627T184635Z_1d248c1a…json"*,
forcing an extra `Read` to scan for an EventGraph-bearing Blueprint to mutate.
Confirmed in source: `PinWright_BlueprintHandlers_List.cpp:248-258` emits
`name`/`path`/`class`/`packagePath`/`parentClass` per row, and `:20-27`
registers only filters + `limit`/`offset` — no `namesOnly`/`fields`.

Severity Low: by the rubric a response-spill that only forces a `Read` is Low,
and `blueprint.list` already ships `limit`+rich filters as a workaround (a tight
`nameFilter`/`pathStartsWith` avoids the spill), so the friction is real but
easily sidestepped — not bumped up despite being a common discovery call.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean BPIR positioned-custom-event round-trip task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `tool_bug`/`F-bpir-multi-self-targets`; 7 RPCs, all `ok`/non-error, zero retries). Friction: the opening discovery call `blueprint.list {limit:40}` returned `outputTooLong` at **27334 chars** (threshold 10000) and spilled to `HttpResponses/20260627T184635Z_1d248c1a…json`, forcing an extra `Read` to scan for an EventGraph-bearing BP to mutate. Verified in source: `PinWright_BlueprintHandlers_List.cpp:248-258` emits `name`/`path`/`class`/`packagePath`/`parentClass` per row; `:20-27` registers filters (`path`/`class`/`tag`/`nameFilter`/`pathStartsWith`/`recursive`) + `offset`/`limit` (default 50) but **no `namesOnly`/`fields` projection** — so even the default limit overflows the enumerate-to-pick shape. Proposed: add a `namesOnly`/`fields` projection (return just `name`+`path`, omit the rest unless asked), mirroring the same lever proposed for `blueprint.graph.get_nodes` in `E-get-nodes-pins-spill-no-projection`. Dedup: ripgrep across OPEN/IN-REVIEW/DONE — no ticket pairs `blueprint.list` with a spill/projection fix; `B-blueprint-list-class-ensure` (DONE, short-class ensure crash), the `*-list-no-limit-spills` family (those lack `limit` entirely — `blueprint.list` already has it), and `E-asset-list-pagination-undocumented-spills` (different method, lever-exists→docs-only) are all distinct.
- `#2-cross-task-evidence-grep-workaround` `OPEN` reporter — SECOND independent recurrence, a larger page **and** a code-fallback workaround that hand-built the proposed projection. From the struggle audit of a clean-process BPIR positioned fork/reconverge round-trip probe (focus `blueprint.compile_bpir`, namespace `blueprint`; 13 RPCs all `ok`/non-error; the iteration's own outcome was the unrelated `B-bpir-fallthrough-reconverge-dropped`). The opening discovery call `blueprint.list {pathStartsWith:"/Game", limit:60}` returned `{outputTooLong:true, characters:43036, threshold:10000}` and spilled the full payload to `Saved/PinWright/HttpResponses/…json`. Notably the caller did not merely `Read` the spill — it dropped out of the MCP entirely and ran a local Bash `grep -oE '\"(path|assetPath|objectPath|packageName|name)\":\"[^\"]*\"'` over the dump file to recover just the name/path pairs for anchor selection, i.e. it manually reconstructed the very names-only projection this ticket proposes. Two independent probes now overflow the enumerate-to-pick shape (27334 chars at limit:40 in #1, 43036 at limit:60 here) — confirming the friction recurs whenever a caller that doesn't yet know which BP it wants cannot pre-filter. Severity unchanged Low (spill-only; tight filters remain a workaround). Dedup: matched this OPEN ticket on rg `blueprint.list`/`outputTooLong`/projection; appended rather than re-filed.
