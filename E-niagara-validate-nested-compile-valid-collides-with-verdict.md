---
id: E-niagara-validate-nested-compile-valid-collides-with-verdict
title: "niagara.validate carries two keys named `valid` that can disagree: the nested compile block's `valid` (UNiagaraSystem::IsValid) next to the authoritative top-level verdict"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, validate, inspect, compile-state, response-shape]
encounters: 1
costly: 0
lastSeen: 2026-10-03T00:00:00+03:00
rice: [1, 2, 1, 1]
priority: 17
---

# Nested `compile.valid` is mistaken for the validate verdict

Split from `E-niagara-validate-compile-state-uninitialized-undocumented` (`#3`, remaining slice named in `#8`).

`niagara.validate` (and `niagara.inspect`'s compile block) returns `compile.valid`, which is
`UNiagaraSystem::IsValid()` / `FVersionedNiagaraEmitterData::IsValid()` / `UNiagaraScript::IsCompilable()`
(`NiagaraDumpBuilder.cpp` `Build*CompileDiagnosticsJson`), beside the authoritative top-level `valid`.
The two booleans can disagree (`compile.valid:false`, top-level `valid:true`). In `#3` of the parent ticket an agent
scanning a spilled 13 KB response for `valid` had to Read an offset of the file to establish which
one was authoritative. `niagara.md` already says "`compile.valid` is not the field to branch on", but
only a caller who read that page knows it.

## What it should do

Give the nested field a name that does not collide, e.g. `compile.engineValid`, or drop it. This is a
response-shape change, so it needs a CHANGELOG **Behaviour change** line, docs in `niagara.md`
(validate and inspect sections), a `niagara_compile.json` check (the authored aspect does not carry
`valid`, so likely no aspect bump), and a test asserting the payload has exactly one `valid`.

## History
- `#1-split` `OPEN` developer — Filed as the remaining slice of `E-niagara-validate-compile-state-uninitialized-undocumented` (`#3` proposal, `#8` handoff) so it survives that ticket closing. Not implemented in batch 5.
