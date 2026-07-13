---
id: E-niagara-search-ops-multiply-keyword-misses-scalar
title: "niagara.graph.search_ops query=\"multiply\" returns only Matrix ops — the scalar Mul needs the abbreviated leaf token, an undocumented keyword/registry mismatch"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, niagara, niagara-graph, search-ops, op-discovery, node-discovery]
encounters: 3
lastSeen: 2026-07-13T11:04:41.6442061+03:00
---

# `search_ops query="multiply"` surfaces only Matrix ops; the scalar `Mul` needs the bare leaf word

`niagara.graph.search_ops` is the discovery RPC an agent reaches for to pick a
valid op `FName` before `niagara.graph.create_node nodeClass="NiagaraNodeOp"`.
But the natural English word for the most common scalar op does not surface it:

- `search_ops { query: "multiply" }` returns **only Matrix ops** (e.g.
  matrix-multiply variants) — the scalar `Numeric::Mul` the caller actually
  wants is **not in that result set**, because the searchable text for the
  scalar op is the abbreviation `Mul`, not the full word "multiply".
- The caller only gets the scalar op by re-querying the **abbreviated leaf**
  token: `search_ops { query: "Mul" }` → `Numeric::Mul`.
- The sibling op behaves better only by luck of spelling: `search_ops
  { query: "add" }` → `Numeric::Add` first try, because "add" *is* the leaf
  spelling.

So the op-search vocabulary (registry leaf names like `Mul`) and the natural
caller vocabulary (the full word "multiply") are **disjoint** for the
abbreviated ops, and nothing in the wiki tells the caller to search the bare
PascalCase leaf token rather than the spelled-out operator name. The result is
a dead-end search on the obvious term followed by a guess-and-retry on the
abbreviation — the same friction class the board already records for
`niagara.search_modules` ("spawn rate" → 0; see
[`E-niagara-standard-stack-recipe-undocumented`](E-niagara-standard-stack-recipe-undocumented.md))
and for MetaSound's `gain` shorthand (see
[`E-metasound-shorthand-search-mismatch`](E-metasound-shorthand-search-mismatch.md)),
but in the Niagara op-registry code path.

## Distinct from the existing tickets

This is **upstream** of, and distinct from,
[`B-niagara-create-op-bare-leaf-pinless`](B-niagara-create-op-bare-leaf-pinless.md):
that ticket (IN-REVIEW) is about which form to **pass to create_node** (the
bare leaf `Mul` vs the registry key `Numeric::Mul`) once you already have the
op — its fix canonicalizes the leaf and documents the `payload.opName`
accepted forms. This ticket is about **finding the op via search_ops in the
first place** — the search-ranking / keyword-mismatch that makes the natural
word "multiply" miss the scalar op. The form-to-pass fix does nothing for the
caller who searched "multiply" and saw only Matrix ops. It is also distinct
from `F-search-api-niagara-graph-nodes` (DONE — that *added* `search_ops`;
the RPC works, the gap here is its keyword behavior + docs).

## Why it's process friction (clean outcome, extra search hop)

The task — build `NS_MuzzleFlash`, add Mul+Add `NiagaraNodeOp`s, wire and
then remove one — completed cleanly, but the op-discovery cost three
`search_ops` calls where two should suffice. Friction note verbatim:

> "search_ops query \"multiply\" returned only Matrix ops (the scalar Mul
> required searching the bare leaf \"Mul\"), a small discoverability snag the
> create_node wiki note pre-warned about."

Call log corroborates: `search_ops query="multiply"` (Matrix only) → then
`search_ops query="add"` (→ `Numeric::Add`) → then `search_ops query="Mul"`
(→ `Numeric::Mul`) — the first search was spent solely because the full word
did not match the abbreviated leaf. Fully recoverable, hence Low severity, but
every author reaching for a scalar multiply (or any abbreviated op:
`Sub`, `Div`, etc.) pays the same dead-end-then-retry tax until the matching
is tolerant or the convention is documented.

## What it should do (docs-first; downstream wiki process, not mine)

Page to improve: `docs/wiki-src/niagara.graph.md` (the `### niagara.graph.search_ops`
discussion, adjacent to the existing `### niagara.graph.create_node`
`payload.opName` note that already explains the leaf-vs-signature forms).

- Add a one-line search-convention note: the op registry stores **abbreviated
  PascalCase leaf** names (`Mul`, `Sub`, `Div`, `Add`), so `search_ops` matches
  the leaf spelling — searching the spelled-out word ("multiply", "subtract",
  "divide") may return only the *Matrix* variants (whose names contain that
  word) and miss the scalar op. Search the **abbreviated leaf token** (`Mul`)
  or a short distinctive substring to surface the scalar `Numeric::*` op.
- Optionally list the half-dozen common abbreviated ops with their full-word
  synonyms (`Mul`≈multiply, `Sub`≈subtract, `Div`≈divide) so the mapping is a
  table lookup, not a guess.

A cheaper code-side alternative (out of scope for this docs ticket, note only):
make `search_ops` keyword matching tolerant of the spelled-out synonyms —
either a small synonym map (`multiply`→`Mul`, `subtract`→`Sub`) or fuzzy/prefix
matching against the leaf — so the natural word surfaces the scalar op without
the caller pre-collapsing it to the abbreviation. Same follow-up shape proposed
for `search_modules` in `E-niagara-standard-stack-recipe-undocumented` `#1`.

## History
- `#3-liveness` `OPEN` reporter — Still observed (prep-friction reconfirm, same `#1` root cause, no new angle). SEED-mode `niagara.graph.list_node_types` task (prototype a `(source * value)` tweak into a brand-new `/Game/FXProto/NS_ParticleMathProto`); Prep's discovery note verbatim: `search_ops query="multiply"` returned only MatrixMultiply/MatrixVectorMultiply, the scalar `Numeric::Mul` surfaced only on `query="mul"`. Identical natural-word-misses-abbreviated-leaf friction as `#1`, different seed. Still Low, fully recoverable. encounters 2 -> 3.
- `#2-additional-divide-empty-and-source` `OPEN` reporter — Additional evidence (re-reproduced at HEAD on a niagara.graph.search_ops seed task: build a Mul NiagaraNodeOp into NS_EQ_Reactive's ParticleUpdate, wire two inputs, compile clean; clean outcome, one extra search hop). Two new angles beyond `#1`: (a) the symptom is **non-uniform across the abbreviated ops** — `#1` lumps "Sub, Div, etc." as paying "the same tax," but replay shows they diverge: `search_ops { query: "subtract" }` → `Numeric::Subtract` (works, because the op's leaf literally IS "Subtract"), whereas `search_ops { query: "divide" }` → `{"results":[],"totalMatches":0}` — a **completely empty** result implying no divide op exists, even though `search_ops { query: "Div" }` → `Numeric::Div` (score 1000). So "divide" is a strictly worse dead-end than "multiply" (which at least returns the Matrix variants): the natural word yields ZERO hits with no hint the op is spelled `Div`. Verbatim replays: `query="multiply"` → `{"results":[{"opName":"MatrixMultiply","signature":"Matrix::MatrixMultiply","category":"Matrix","score":100},{"opName":"MatrixVectorMultiply","signature":"Matrix::MatrixVectorMultiply","category":"Matrix","score":100}],"totalMatches":2}`; `query="divide"` → `{"results":[],"totalMatches":0}`. (b) **Guilty source line** — the code-side root cause is confirmed in `Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraSearchHandler.cpp:236-237`, where the handler calls `ScoreOpMatch` with the Alternate and Keywords args **hardcoded empty**: `const int32 Score = NiagaraSearch::ScoreOpMatch(Query, OpName, /*Alternate*/ FString(), Category, /*Keywords*/ FString());`. `ScoreOpMatch` supports an "AlternateSearchName" (score 200) and "Keywords contains" (score 50) path, but search_ops never populates either — so matching is pure name/category substring, and the wiki's documented "Search the Niagara op registry by name/category/**keywords**" (`niagara.graph.search_ops.md`) over-claims a keywords facet that is never fed any data. Any code-side fix (synonym map or populating the catalog's keywords/alternate) lives at that call-site. Still Low, still fully recoverable. Culprit is the seed itself (`niagara.graph.search_ops`).
- `#1-initial-audit` `OPEN` reporter — PROCESS/docs friction from a clean `niagara.graph.remove_node` fuzz task (build NS_MuzzleFlash, add Mul+Add NiagaraNodeOps, wire Mul.Result→Add.A, remove the Add node; outcome clean, no retries/python fallback). Op discovery cost 3 `search_ops` calls because the natural full word missed the scalar op: `query="multiply"` returned **only Matrix ops** (scalar `Numeric::Mul` absent), then `query="add"`→`Numeric::Add`, then the abbreviated `query="Mul"`→`Numeric::Mul`. The op-search vocabulary (abbreviated leaf `Mul`) and the caller's natural vocabulary (full word "multiply") are disjoint, and the wiki never says to search the bare PascalCase leaf. Same friction class as `E-niagara-standard-stack-recipe-undocumented` (search_modules "spawn rate"→0) and `E-metasound-shorthand-search-mismatch` (gain shorthand vs Multiply registry name), in the Niagara op-registry path. Distinct from `B-niagara-create-op-bare-leaf-pinless` (which form to *pass to create_node* — already fixed/documented; this is *finding the op via search_ops*) and from DONE `F-search-api-niagara-graph-nodes` (added the RPC; the gap is its keyword behavior + docs). Low severity — fully recoverable, one extra dead-end search per abbreviated op. Page to improve: `docs/wiki-src/niagara.graph.md` (search_ops convention note + optional abbreviated-op synonym table); code-side synonym/fuzzy matching noted as a cheaper alternative, out of docs scope.
