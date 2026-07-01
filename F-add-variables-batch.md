---
id: F-add-variables-batch
title: "No batch blueprint.add_variables — authoring N member variables costs N round-trips"
status: OPEN
severity: Low
category: feature
tags: [blueprint, add_variable, batch]
encounters: 1
lastSeen: 2026-07-01T09:41:00.8756290+03:00
---

# No batch variable authoring on `blueprint.*`

## What's missing
There is no way to add multiple member variables in one call. Authoring N variables
requires N separate `blueprint.add_variable` round-trips, each a full RPC that also
re-emits the growing blueprint snapshot (see `E-add-variable-full-snapshot-spill`).

## What it should do
A single `blueprint.add_variables` accepting an array of
`{name, type, category, defaultValue, …}` specs, adding them all and returning one compact
confirmation (added names/types + per-spec success). This mirrors the existing
batch-convenience tickets on other namespaces (`F-batch-pin-defaults`,
`F-add-mapping-batch-keys`, `F-console-batch-get-cvar-values`, etc.).

## Evidence
Struggle audit of a clean at-scale `blueprint.compile_bpir` task (focus
`blueprint.compile_bpir`, namespace `blueprint`, outcome `done`/GREEN, 28 RPC calls,
transcript `agent-aa455a45b45a94d68.jsonl`, BP `/Game/BP_ScaleStress`). The agent
explicitly intended a batch — SAY (trace line 637): "Now add all member variables in one
batch." — but, no batch method existing, fanned out to 9 separate `blueprint.add_variable`
calls (trace lines 638-652) for the 9 variables the scale task required, each a full
round-trip.

severity rationale: impact=N extra calls with a workaround (fan-out) × reach=common
BP-authoring path but only bites at multi-variable scale -> Low.

## Distinct from
- `E-add-variable-full-snapshot-spill` — the per-call response bloat (a batch method also
  helps by collapsing N echoes into one, but the response-shape fix is independent).
- Other `add_variable` tickets (name/type/format/category/default) — single-call facets,
  not the missing batch capability.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean at-scale `blueprint.compile_bpir` task (transcript `agent-aa455a45b45a94d68.jsonl`, BP `/Game/BP_ScaleStress`, 28 RPC calls, GREEN). Agent stated batch intent ("Now add all member variables in one batch", trace line 637) but had to issue 9 separate `blueprint.add_variable` calls (lines 638-652) because no batch method exists. Proposed: `blueprint.add_variables` taking an array of variable specs, returning one compact confirmation. Dedup: ripgrep across OPEN/IN-REVIEW/DONE — batch-convenience tickets exist on other namespaces (`F-batch-pin-defaults`, `F-add-mapping-batch-keys`, `F-console-batch-get-cvar-values`) but none for `blueprint` member variables. Genuinely new. severity Low (workaround = N fan-out calls; common path but only at multi-variable scale).
