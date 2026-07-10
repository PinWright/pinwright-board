---
id: E-niagara-create-node-input-type-unverifiable
title: "niagara.graph.create_node NiagaraNodeInput accepts payload.inputType='float' and returns success, but the created input node's type is not verifiable on readback (downstream op pins read as generic Numeric)"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, niagara-graph, create-node, readback, node-type, docs]
encounters: 1
lastSeen: 2026-07-11T01:58:10.9079381+03:00
---

# `create_node` NiagaraNodeInput `inputType` is applied blind — no readback confirms the typed pin

`niagara.graph.create_node nodeClass="NiagaraNodeInput"` accepts a
`payload.inputType` (documented on the DONE feature ticket
`F-niagara-graph-create-node`, payload field `inputType?: string // NiagaraNodeInput`)
and returns `success:true`, but a caller has **no readback that confirms the
created input node actually carries the requested type**. After creating two
`NiagaraNodeInput` nodes with `payload.inputType='float'` and wiring them into a
`Numeric::Mul` op's A/B pins, the post-save `niagara.graph.get` readback shows
the resolved `Multiply` node's A/B input pins as category **`Numeric`
(NiagaraNumeric)** — the op's numeric-wildcard pin category — not the `float`
that was requested. Because the readback surfaces the consuming op's
wildcard pins rather than the `NiagaraNodeInput`'s own declared variable type,
the `inputType` payload field **looks ignored even though it may have applied
correctly** — there is no observable signal either way.

## Why it's process friction (clean outcome, unverifiable param)

The task — inject a small `(a * b)` math tweak into `NS_EQ_Reactive`'s emitter
`EQ` ParticleUpdate graph — completed cleanly (`compiled:true`,
`niagara.validate valid:true`, saved). The wildcard pins connected fine and the
op resolved, so this did **not** block the edit. But the caller was left
uncertain whether `inputType='float'` took effect. Friction note verbatim:

> "NiagaraNodeInput pins read back as generic NiagaraNumeric rather than the
> float I passed via payload.inputType, but the wires still connected and the
> op resolved cleanly, so no real blocker."

This is a `success:true` with a payload field whose effect is invisible on every
available readback — a discoverability gap, not a proven defect.

## What it should do (hunch — Audit to sanity-check which is true)

Two possibilities, and the fix differs per which holds:

- If `inputType` **is** honored (the input variable is typed `float`) but only
  the downstream op's numeric-wildcard pins are shown, then this is a **docs /
  readback-visibility** gap: `niagara.graph.get` (or a single-node projection)
  should surface the `NiagaraNodeInput`'s own declared variable type, and the
  wiki should note that an op's A/B pins stay `Numeric` regardless of the typed
  input feeding them, so a `Numeric` readback does **not** mean `inputType` was
  dropped.
- If `inputType` **is** silently ignored (input node created untyped/numeric),
  then `create_node` is dropping a supplied payload field while reporting
  success — a misleading-success ergonomic bug that should either honor the
  field or reject an unhonored value.

Page to improve (docs path): `docs/wiki-src/niagara.graph.create_node.md` — for
`nodeClass="NiagaraNodeInput"`, document what `inputType` does, whether the
resulting input pin is concretely typed or a numeric wildcard resolved at
compile, and how to read the created node's type back (since a downstream-op
`graph.get` shows the op's wildcard pins, not the input's declared type).

## Distinct from

- `E-niagara-input-schema-readback` (IN-REVIEW) — typed per-input schema for
  placed **module stack** inputs (`niagara.inspect` includeStack /
  `set_module_input`), a different surface; this is a **graph** `create_node`
  input node whose type isn't verifiable via `niagara.graph.get`.
- `F-niagara-graph-create-node` (DONE) — the feature that *added* `create_node`
  and documents the `inputType?` payload field; the RPC works and the node is
  created — the gap here is that the field's effect is unobservable on readback.

## Severity rationale

severity rationale: impact=discoverability/naming (works but non-obvious — a
supplied param's effect is invisible on readback, forcing a caller to trust an
unverifiable success) × reach=rare (raw Niagara script-graph node authoring, not
an every-session path) -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the clean/`done` `niagara.graph.search_ops` fuzz task (focus `niagara.graph.search_ops`, namespace `niagara.graph`, outcome ergo — the judge filed the orthogonal search-keyword ticket `E-niagara-search-ops-multiply-keyword-misses-scalar`; this is the distinct `create_node` PROCESS angle the judge did not touch). Task: add a `Numeric::Mul` op + two `NiagaraNodeInput` nodes to `NS_EQ_Reactive` emitter `EQ` ParticleUpdate, wire A/B, compile+validate clean. Both input nodes were created with `payload.inputType='float'` and returned success, but the post-save `niagara.graph.get` readback showed the Mul node's A/B pins as category `Numeric` (NiagaraNumeric), not `float` — the input node's declared type is not surfaced by any readback, so `inputType` looks ignored even though it may have applied. Framed as a hunch: Audit should sanity-check whether `create_node` honors `inputType` for NiagaraNodeInput (then it's a docs/readback-visibility gap on `docs/wiki-src/niagara.graph.create_node.md`) or silently drops it (then it's a misleading-success ergonomic bug). Low severity — did not block the edit (op compiled/validated/saved clean), fully recoverable, rare authoring path. Dedup: ripgrep across OPEN/closed found no `create_node`/`NiagaraNodeInput` input-type-readback ticket; `E-niagara-input-schema-readback` (module stack inputs, different surface) and `F-niagara-graph-create-node` (the DONE feature that added the RPC) are distinct.
