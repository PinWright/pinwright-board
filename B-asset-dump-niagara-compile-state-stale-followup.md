---
id: B-asset-dump-niagara-compile-state-stale-followup
title: "Niagara compile-state stale fix incomplete — 80/81 NiagaraSystems still report NCS_Unknown + readyToRun=false; NiagaraEmitters lack the deferred-on-load field entirely"
status: DONE
severity: High
category: bug
tags: [asset-dump, niagara, compile-state, regression]
---

# `B-asset-dump-niagara-compile-state-stale` fix did not take

Follow-up to `B-asset-dump-niagara-compile-state-stale` (marked DONE).
The 2026-05-04 re-dump shows the original symptom is largely
unchanged.

## Observations

**NiagaraSystem (81 assets):**
- 80 / 81 still report `readyToRun: false`.
- Of the 1,286 total scripts across systems: 1,282 are
  `NCS_Unknown`, only 4 are `NCS_UpToDate` (all on `FXE_Trail`).
- The `compileDeferredOnLoad: false` field IS present (the fix
  shipped that part), but the sentinel pairing is wrong: the
  field reads `false` while every script still reports
  `NCS_Unknown` and the system reports `readyToRun: false`. There
  is no `COMPILE_DEFERRED_ON_LOAD` issue entry in `issues[]` to
  flag the contradiction (`issues: []` in every file).

**NiagaraEmitter (44 assets):**
- All 44 have `readyToRun: true`.
- All 44 have every script reporting `NCS_Unknown` (176 scripts
  total).
- The `compileDeferredOnLoad` field is **entirely absent** —
  the fix only touched the NiagaraSystem schema, not the
  NiagaraEmitter schema.
- `readyToRun: true` + every script `NCS_Unknown` is a
  contradictory state.

## Two distinct sub-bugs

1. **NiagaraSystem dump captures pre-compile state regardless of
   the deferred-on-load CVar.** The fix added the
   `compileDeferredOnLoad` field but the predicate that drives
   `NCS_Unknown` is not actually about the deferred CVar — it's
   that the dump runs in batch and reads compile state before
   the editor has triggered any compilation. The fix added a
   field but didn't change the underlying behavior.

2. **NiagaraEmitter schema missing the field entirely** — the
   fix didn't extend to the standalone-emitter dump path.

## Fix (proposed)

Pick one of the three options from the original ticket:
- (preferred) Trigger a synchronous compile per
  NiagaraSystem/NiagaraEmitter before reading compile state in
  the dumper. Mirrors what the editor does on user open.
- OR omit `compileStatus` and `readyToRun` when the asset has
  not been compiled in the current session, and emit a
  `compile_state_uninitialized` issue entry instead.
- OR document that `niagara_compile.json` reflects asset-load
  state, not editor-compiled state, and stop emitting it as a
  signal of correctness.

For the NiagaraEmitter schema gap: extend the same shape used
for NiagaraSystem (`compileDeferredOnLoad` plus the chosen
issue-entry approach).

## Repro

- `App\App\FXE_Trail\niagara_compile.json` — `readyToRun: false`,
  `compileDeferredOnLoad: false`, all scripts `NCS_Unknown`,
  `issues: []`. (Wait — FXE_Trail is the one with `readyToRun: true`
  per the audit; sample shows the inconsistency.)
- `Game\Effects\Particles\Environmental\Emitters\NE_BoneBasedEmission\niagara_compile.json`
  — `readyToRun: true` but every script `NCS_Unknown`,
  `compileDeferredOnLoad` field absent.

## History
- `#1-followup-to-stale-fix` `OPEN` reporter — Fresh 2026-05-04 re-dump after the IN-REVIEW→DONE transition on `B-asset-dump-niagara-compile-state-stale` shows the symptom is essentially unchanged. 80/81 NiagaraSystems still report all-NCS_Unknown + readyToRun=false; 44 NiagaraEmitters report a contradictory readyToRun=true + all-NCS_Unknown and lack the `compileDeferredOnLoad` field entirely. The shipped fix added a metadata field on NiagaraSystem but did not change the underlying "dump captures pre-compile state" behavior, and the schema fix was not extended to NiagaraEmitter. Filing as a follow-up rather than re-opening per the no-skip-returned-tasks convention.
- `#2-uninitialized-issue-and-emitter-parity` `IN-REVIEW` developer — Per-script and system-level null-out for NCS_Unknown state in NiagaraDumpBuilder.cpp; new COMPILE_STATE_UNINITIALIZED info issue on both BuildCompileJson and BuildEmitterCompileJson; compileDeferredOnLoad parity field added to BuildEmitterCompileJson; regression tests FNiagaraDumpUninitializedSystemTest and FNiagaraDumpUninitializedEmitterTest added in TestNiagaraDumpCompileDeferred.cpp. Dumper stays read-only — does not trigger any compile.
- `#3-verify-uninitialized-state` `DONE` tester — Verified: ran `asset.dump` on `/App/App/FXE_Trail` and `/Game/Effects/Particles/Environmental/Emitters/NE_BoneBasedEmission`; both `niagara_compile.json` files include `compileDeferredOnLoad: false`, use null script compile statuses instead of `NCS_Unknown`, and emit `COMPILE_STATE_UNINITIALIZED`.
