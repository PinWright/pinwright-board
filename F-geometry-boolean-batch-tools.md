---
id: F-geometry-boolean-batch-tools
title: "No batch boolean op — drilling N identical holes / cutting N tools from one target is N create + N boolean_subtract calls (8 for four corner bolt holes), each tool restated singly"
status: OPEN
severity: Low
category: feature
tags: [geometry, boolean_subtract, boolean_union, boolean_intersection, batch, pipeline, drill, holes, docs]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# Subtracting N tool meshes from one target is 2N sequential calls with no batch entry point

`geometry.boolean_subtract` (and its `boolean_union` / `boolean_intersection` /
`difference` siblings) takes exactly **one** `targetActor` + **one** `toolActor`
per call (`BooleanHandler.cpp:204-206,217-219`; `RPC_PARAM_REQ("toolActor",
"string", ...)` — a single string, no array form). The recurring authoring
intent "cut these N tools out of this one plate" — e.g. **drill four corner bolt
holes** into a mounting flange, or punch a row of windows into a wall — therefore
decomposes into a fixed `N × create_<primitive>` + `N × boolean_subtract` chain
against the **same target actor**, with the target restated on every cut and the
tool actors created/named one at a time. There is no composite verb that takes
the target once plus an array of tool actors (or an array of inline tool specs)
and applies them under one transaction.

## Why this is friction (not just normal granularity)

This is the same friction class already recognized on the board for other
`geometry` surfaces — `F-geometry-uv-prep-pipeline-batch` (the unwrap→project→
transform→pack pipeline is 4 same-actor calls) and `F-geometry-lod-settings-batch`
(the LOD ladder is N same-surface calls) — and the cross-namespace batch
precedent (`F-batch-pin-defaults`). Here the single conceptual step "drill four
bolt holes" costs **eight** RPCs (4 create_cylinder + 4 boolean_subtract), with
`targetActor:FlangePlate` repeated on all four cuts and four separately-named
throwaway cutter actors (`Bolt_PP/PN/NP/NN`) that exist only to be subtracted and
discarded (`keepTool:false`). The boilerplate scales linearly with hole count and
is paid every time a part has a repeated cut pattern (bolt circles, perforations,
window grids) — the single most common reason to run several booleans against one
target in a row.

This is a clean-outcome PROCESS finding: every call in the task succeeded first
try with no retries, errors, workarounds, or `python.execute` fallback. The
friction is the call *count* and the repeated-target boilerplate on an otherwise
clean run, surfaced by the struggle audit (the per-finding judge filed
`B-geometry-convert-static-mesh-no-disk-write` for the unrelated bake-persistence
bug; this batch-call gap is a distinct PROCESS angle).

## What it should do

Pick one of:
1. **Composite handler (preferred, matches the `F-geometry-*-batch` /
   `F-batch-pin-defaults` precedent):** add a `geometry.boolean_subtract_many`
   (or a `tools: [...]` array on the existing verb) that takes `targetActor`
   **once** plus an ordered `tools: ["Bolt_PP", "Bolt_PN", ...]` array (and an
   optional shared `keepTool`), applying each tool to the target in sequence
   under one transaction with a per-tool `results` array. The singular
   `boolean_subtract` stays for fine-grained use. This collapses "drill four
   holes" from 4 boolean calls to 1. (A richer variant could also accept inline
   tool specs — `tools:[{shape:"cylinder", radius:12, height:60, location:{...}}]`
   — to fold the N `create_cylinder` calls in too, taking the whole four-hole
   pattern from 8 calls to 1.)
2. **Docs-only (minimum):** note on the `geometry` overlay
   (`docs/wiki-src/geometry.md`) that boolean ops are one-tool-per-call and that a
   repeated cut pattern is N create + N subtract against the same target, so an
   agent sizes the call budget correctly and names cutter actors systematically.
   (The downstream wiki edit is the wiki-authoring process, not this audit's job;
   this ticket names the page.)

## Friction evidence (this task — geometry "PipeFlange" mounting bracket, 31 calls, outcome tool_bug)

Story: build a flange plate, subtract a central bore + **four corner bolt holes**,
bevel, collision, bake. The bolt-hole half of the call log is the fixed
`4 × create_cylinder` + `4 × boolean_subtract` pattern, every subtract on the same
`FlangePlate` target, all `ok:true`:

- `geometry.create_cylinder {Bolt_PP r12 h60 @+70+70}` → ok
- `geometry.create_cylinder {Bolt_PN r12 h60 @+70-70}` → ok
- `geometry.create_cylinder {Bolt_NP r12 h60 @-70+70}` → ok
- `geometry.create_cylinder {Bolt_NN r12 h60 @-70-70}` → ok
- `geometry.boolean_subtract {Bolt_PP from FlangePlate keepTool:false}` → ok
- `geometry.boolean_subtract {Bolt_PN from FlangePlate keepTool:false}` → ok
- `geometry.boolean_subtract {Bolt_NP from FlangePlate keepTool:false}` → ok
- `geometry.boolean_subtract {Bolt_NN from FlangePlate keepTool:false}` → ok

The agent's friction note was *"none — wiki was complete"* — so nothing errored;
this is pure call-count / repeated-target overhead on an otherwise clean run. A
`geometry.boolean_subtract_many` (or `tools:[]` array) would collapse the four
cuts to one declarative call, and the inline-spec variant would fold the four
cutter creations in too.

## Dedup

Distinct from `B-boolean-subtract-ignores-tool-offset` (the *boolean* silently
no-ops on an offset tool — a correctness bug, since fixed) and from
`B-geometry-convert-static-mesh-no-disk-write` (the bake-persistence bug the judge
filed for this task) — both are correctness/persistence, not call-count. Same
batch-friction *class* as `F-geometry-uv-prep-pipeline-batch` and
`F-geometry-lod-settings-batch` but on the boolean-cut surface, which neither
covers (those are the UV-prep and LOD-ladder surfaces); no existing
boolean/batch-cut ticket on the board.

## Docs page to improve

`docs/wiki-src/geometry.md` — if the docs-only floor is taken instead of the
composite verb, add a short note that boolean ops are one-tool-per-call and a
repeated cut pattern (bolt circle / window grid) is N create + N subtract against
the same target.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry` "PipeFlange"
  mounting-bracket build (31 calls, outcome tool_bug; judge filed
  `B-geometry-convert-static-mesh-no-disk-write` for the bake-persistence bug).
  Distinct PROCESS/batch finding: `geometry.boolean_subtract` takes one
  `targetActor` + one `toolActor` per call (`BooleanHandler.cpp:204-206,217-219`,
  single-string `toolActor`, no array), so the single intent "drill four corner
  bolt holes" cost 8 RPCs (4 create_cylinder + 4 boolean_subtract), with
  `targetActor:FlangePlate` restated on every cut and four throwaway
  `keepTool:false` cutter actors. No retries/errors/fallbacks; agent friction note
  "none — wiki was complete". Proposed: add a `geometry.boolean_subtract_many` (or
  a `tools:[]` array on the existing verb) that takes the target once plus an
  ordered tool array under one transaction (optionally accepting inline tool specs
  to fold the N creates in too), mirroring the `F-geometry-uv-prep-pipeline-batch`
  / `F-geometry-lod-settings-batch` / `F-batch-pin-defaults` batch precedent; or at
  minimum a docs note on `docs/wiki-src/geometry.md`. Dedup: same batch-friction
  class as the UV-prep and LOD-ladder batch tickets but on the boolean-cut surface
  (neither covers it); distinct from the boolean-offset bug and the convert
  disk-write bug; no existing boolean/batch-cut ticket on the board.
