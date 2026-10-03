---
id: B-niagara-validate-stateless-emitter-reported-missing
title: "niagara.validate / compile_status treat a stateless (lightweight) emitter as broken: EMITTER_DATA_MISSING error plus four phantom script entries that keep scriptCompileCheck unverified"
status: OPEN
severity: Medium
category: bug
tags: [niagara, validate, compile-status, compile-state, stateless-emitter]
encounters: 1
costly: 0
lastSeen: 2026-10-03T00:00:00+03:00
rice: [1, 2, 0.8, 2]
priority: 7
---

# A system with a stateless emitter fails `niagara.validate` and can never pass `scriptCompileCheck`

Source-derived (found in review of `E-niagara-validate-compile-state-uninitialized-undocumented` `#8`), not yet reproduced live.

`FNiagaraEmitterHandle::GetEmitterData()` returns `nullptr` for any handle whose `EmitterMode` is `ENiagaraEmitterMode::Stateless` (UE 5.4+, engine `NiagaraEmitterHandle.cpp:243-246`). The engine's compile skips those handles (`NiagaraSystem.cpp:3931`), so they have no VM scripts. PinWright's compile builders treat that `nullptr` as a corrupt emitter:

- `NiagaraDumpBuilder.cpp` `BuildSystemAuthoredIssues` (the `if (!EmitterData)` branch, ~:1523) raises an **error** `EMITTER_DATA_MISSING` ("Emitter handle has no versioned emitter data"). `niagara.validate` copies compile-block issues into `errors[]` (`NiagaraInspectHandler.cpp` ~:151-172), so a healthy system with a lightweight emitter validates `valid: false`.
- `AddEmitterCompileEntries` still appends four entries (EmitterSpawn/EmitterUpdate/ParticleSpawn/ParticleUpdate) built from `nullptr` (`{present:false}`, no `compileStatus`, no `compiledIntoSystemScripts` marker). `ReadCompileVerdict` (`NiagaraCompileVerdict.cpp`) counts them as scripts that compile on their own but never as terminal, so `scriptCompileCheck` / `niagara.compile_status` can never reach `passed` / `completed` for such a system, even after `niagara.compile`.

## What it should do

Branch on `Handle.GetEmitterMode() == ENiagaraEmitterMode::Stateless` (guarded `UE_VERSION_NEWER_THAN_OR_EQUAL(5, 4, 0)`): no `EMITTER_DATA_MISSING`, no script entries (or entries marked as having no VM scripts and skipped by the verdict fold). Keep `EMITTER_DATA_MISSING` for a Standard handle with no data. Test: a system with one stateless emitter handle validates with no `EMITTER_DATA_MISSING` and, after a forced compile, `scriptCompileCheck: "passed"`.

## History
- `#1-found-in-review` `OPEN` reviewer — Found while reviewing `E-niagara-validate-compile-state-uninitialized-undocumented` `#8`. That fix folds the emitter spawn/update entries out of the verdict, which makes `passed` reachable for standard emitters only; a stateless handle's four `present:false` entries still withhold it, and its `EMITTER_DATA_MISSING` error fails `niagara.validate` outright. Evidence is source reading of the 5.8 engine and the current plugin tree; no live repro yet. Dedup: no board ticket mentions stateless/lightweight emitters or `EMITTER_DATA_MISSING` on a validate path.
