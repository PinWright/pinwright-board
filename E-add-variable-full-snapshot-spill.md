---
id: E-add-variable-full-snapshot-spill
title: "blueprint.add_variable echoes the whole growing blueprint snapshot every call — crosses the 10KB inline limit at scale, forcing a spill Read"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, add_variable, response-size, spill, http-spill]
encounters: 1
lastSeen: 2026-07-01T09:41:00.8756290+03:00
---

# `blueprint.add_variable` returns the entire blueprint snapshot on every call

## What's awkward
Every `blueprint.add_variable` response re-serializes the ENTIRE class — all existing
`variables[]`, all `functions[]`, per-variable metadata, plus a duplicate top-level
`variable` object for the one just added. Response size therefore grows linearly with the
variable count already on the Blueprint. On a scale-authoring task that added a 9-variable
batch, the 7th-9th `add_variable` responses crossed the 10,000-char inline limit three
times in a row and spilled to `Saved/EditorAutomation/HttpResponses/…json`, costing an
extra file Read just to confirm the last variables landed. The individual add succeeded
each time — the payload bloat is pure overhead.

## What it should do
Default to a compact confirmation: the added variable's name/type + success (mirroring the
minimal-echo pattern other mutators use), with the full blueprint snapshot behind an
opt-in flag (e.g. `includeSnapshot: true`). A single-variable add should not return the
whole class.

## Evidence
Struggle audit of a clean at-scale `blueprint.compile_bpir` task (focus
`blueprint.compile_bpir`, namespace `blueprint`, outcome `done`/GREEN, 28 RPC calls,
transcript `agent-aa455a45b45a94d68.jsonl`, BP `/Game/BP_ScaleStress`). The CallAnalyzer
(ground-truth call-trace, strongest signal) flagged it: trace lines 665-667 show three
consecutive `add_variable` responses reporting `Response exceeds display limit (10830 /
11932 / 13060 chars …)` with the full payload written to a `HttpResponses/…json` spill
file; line 669 is the forced Read of the dumped payload to confirm success. The spill grew
monotonically (10830 -> 11932 -> 13060 chars) as each new variable enlarged the echoed
snapshot.

severity rationale: impact=response-spill that only forces a Read × reach=common method
but the spill only triggers past ~7 variables (not every session) -> Low.

## Distinct from
- `E-add-variable-name-vs-variablename` / `E-add-variable-type-format` /
  `E-add-variable-category-param-undocumented` / `B-add-variable-default-value-ignored` —
  param naming, value format, category, and default-echo facets of the same method; none
  concerns the full-snapshot response-size bloat.
- `E-http-response-spill` (DONE) — that added the generic HTTP spill *fallback*; this asks
  the `add_variable` handler to not produce an oversized body in the first place.
- `F-add-variables-batch` — the missing batch capability (a batch method would also
  collapse N growing echoes into one compact response); this ticket is the per-call
  response-shape ergonomic, independent of batching.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean at-scale `blueprint.compile_bpir` task (transcript `agent-aa455a45b45a94d68.jsonl`, BP `/Game/BP_ScaleStress`, 28 RPC calls, GREEN, no correctness defect on the focus method). Adding 9 member variables, the 7th-9th `add_variable` responses crossed the 10KB inline limit (10830/11932/13060 chars, trace lines 665-667) because each echoes the full growing blueprint snapshot (all variables[] + functions[] + metadata + a duplicate top-level `variable`); the payload spilled to `HttpResponses/…json` and forced an extra Read (line 669) to confirm the last adds landed. Proposed: compact success confirmation by default, full snapshot behind an opt-in flag. Dedup: ripgrep across OPEN/IN-REVIEW/DONE — the other `add_variable` tickets cover param naming/format/category/default facets, none the snapshot bloat; `E-http-response-spill` (DONE) is the generic transport fallback, not a per-handler fix. Genuinely new. severity Low (response-spill forcing a Read; common method but spill only past ~7 vars).
