---
id: B-asset-dump-niagara-compile-state-stale
title: "asset.dump records niagara_compile.json with NCS_Unknown / readyToRun=false on every NiagaraSystem"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, niagara, compile-state]
---

# `niagara_compile.json` records uninitialized state for every NiagaraSystem

In a 124-asset Niagara dump (80 `NiagaraSystem` + 44 `NiagaraEmitter`),
every `NiagaraSystem`'s `niagara_compile.json` reports:
- `readyToRun: false`
- `hasOutstandingCompilationRequests: true`
- every script: `compileStatus: NCS_Unknown`

This is across all 80 systems, including ones that have shipped
content. Standalone `NiagaraEmitter` assets are not affected
(`readyToRun` not present in their schema).

The pattern indicates the dump is reading the system's compile state
before the editor has compiled it in the current session. Because the
dump runs in batch and doesn't trigger compilation per asset, every
system ends up reporting the uninitialized state. The downstream
consumer of the cache then sees "every Niagara system is broken,"
which is wrong.

**Repro:** `asset.dump_folder("/App", { "recursive": true })`,
inspect any `niagara_compile.json` under e.g.
`App\App\FXE_Trail\niagara_compile.json` — 6/6 scripts `NCS_Unknown`,
`readyToRun: false`.

**Fix (implemented):** UE 5.6 ships `fx.Niagara.OnDemandCompile` defaulted on, deferring compile until Niagara editor open or FX-component spawn. Forcing sync compile per system would stall the editor 1–3 minutes on an 80-system batch dump and burn the dump's tick budget. Right fix is schema-additive: read the CVar, write a `compileDeferredOnLoad` boolean to `niagara_compile.json`, and append an info-severity issue with code `COMPILE_DEFERRED_ON_LOAD` so downstream consumers can distinguish "not yet compiled this session" from "actually broken." No engine state mutated.

## History
- `#1-initial-audit` `OPEN` reporter — All 80 cached `niagara_compile.json` files for NiagaraSystem assets show `readyToRun: false` + every script `NCS_Unknown`. Standalone NiagaraEmitter dumps unaffected. Suggests dump captures pre-compile state in batch mode.
- `#2-cvar-aware-diagnostic` `IN-REVIEW` developer — Added `compileDeferredOnLoad` boolean and `COMPILE_DEFERRED_ON_LOAD` info-severity issue to `BuildCompileJson` in `NiagaraDumpBuilder.cpp`, gated on `fx.Niagara.OnDemandCompile`. Rejected sync-compile path (1–3 min editor stall on 80-system batch). Test: `EditorAutomationRpcGateway.Niagara.DumpCompile.DeferredFlag`.
- `#3-verified-deferred-flag-emitted` `DONE` tester — Verified: `asset.dump /App/App/FXE_Trail` produced `niagara_compile.json` containing `compileDeferredOnLoad: false` field (schema-additive contract satisfied); `issues: []` is consistent with the gate (no `COMPILE_DEFERRED_ON_LOAD` emitted because CVar evaluated false in this session).
