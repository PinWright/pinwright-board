---
id: E-asset-dump-niagara-compile-issue-promote-to-meta
title: "Niagara deferred/uninitialized compile state buried in issues[] — promote to top-level meta.json status"
status: DONE
severity: Low
category: ergonomic
tags: [asset-dump, niagara, compile-state, meta, ergonomics]
---

# Promote Niagara deferred/uninitialized compile-state signal to `meta.json`

Follow-up ergonomic gap to `B-asset-dump-niagara-compile-state-stale` and
its `-followup` ticket (both DONE). Those tickets fixed the correctness
problem: the dumper now emits `compileDeferredOnLoad` and a
`COMPILE_STATE_UNINITIALIZED` info issue inside `niagara_compile.json`
when the editor hasn't compiled the system this session, and the dumper
stays read-only (doesn't self-trigger compile).

The remaining gap is purely ergonomic. A downstream consumer that wants
to filter "which Niagara assets have a usable compile-state dump" has to:

1. Open the asset folder.
2. Read `niagara_compile.json`.
3. Walk `issues[]` looking for the `COMPILE_STATE_UNINITIALIZED` code,
   OR check the `compileDeferredOnLoad` boolean.

`meta.json` is the natural top-level summary surface — it already carries
asset class, parent class, skip status, etc. A consumer scanning a dump
tree shouldn't have to open a sidecar to learn "this asset's compile
state isn't real, you need to open it in the editor and re-dump."

## Proposed change

Add a small top-level field to the NiagaraSystem / NiagaraEmitter
`meta.json` (mirrors the existing `skipped` / `skipReason` pattern):

```json
{
  "...": "...",
  "compileStateAvailable": false,
  "compileStateReason": "deferred-on-load"   // or "uninitialized"
}
```

When the dumper emits `COMPILE_STATE_UNINITIALIZED` or sets
`compileDeferredOnLoad: true` in `niagara_compile.json`, mirror that
into `meta.json` as `compileStateAvailable: false` plus a short reason
code. When the compile state is real, either omit both fields or set
`compileStateAvailable: true`.

This is additive — `niagara_compile.json` keeps the detail; `meta.json`
just gains a glanceable summary so consumers can branch without opening
the sidecar.

## Workaround

Read `niagara_compile.json` directly and check `compileDeferredOnLoad`
or scan `issues[]` for `COMPILE_STATE_UNINITIALIZED`.

## History
- `#1-meta-promotion-request` `OPEN` reporter — Two NiagaraSystem assets in the dump (`FX_Popping_the_Balloon`, `FX_Popping_the_Balloon_Red`) carry the deferred/uninitialized signal only inside `niagara_compile.json`'s `issues[]` — a consumer doing a tree-walk has to open the sidecar to learn the compile state isn't real. Propose mirroring the signal as a `compileStateAvailable` + `compileStateReason` pair on `meta.json` (additive, alongside the existing `skipped`/`skipReason` pattern). Correctness was already addressed by `B-asset-dump-niagara-compile-state-stale[-followup]`; this is the surface-it-at-the-top ergonomic follow-up.
- `#2-compile-state-promoted-to-meta` `IN-REVIEW` developer — Added `FCompileStateSummary` struct plus `GetSystemCompileStateSummary` / `GetEmitterCompileStateSummary` to `Handlers/Niagara/NiagaraDumpBuilder.{h,cpp}`. Helpers return `{ bAvailable: false, Reason: "deferred-on-load" }` when the UE 5.6 `fx.Niagara.OnDemandCompile` CVar is on, `{ false, "uninitialized" }` when any script reports `NCS_Unknown` via `ScriptsHaveUninitializedCompileStatus`, otherwise `{ true, "" }`. Reuses existing `BuildSystemCompileScriptArray` / `BuildEmitterCompileScriptArray` and `NiagaraDecompileHelpers::IsNiagaraOnDemandCompileEnabled`. `BuildAllFilesForAsset` in `Handlers/Asset/AssetDumpHandler.cpp` now calls the summary on `UNiagaraSystem` / `UNiagaraEmitter` after dispatch and a new `AttachNiagaraCompileStateToMeta` helper writes `compileStateAvailable: false` + `compileStateReason: "<reason>"` to `meta.json`. Both fields are omitted when state is available (additive policy). Bundled into the schema-v6 bump in `Utils/AssetDumpBuilder.cpp`.
- `#3-skip-editor-offline` `SKIP` tester — Editor MCP gateway unreachable (HTTP 19880 connection refused), so `asset.dump` couldn't be run live against `FX_Popping_the_Balloon` to read the regenerated `meta.json`. Static review of the diff confirms the wiring matches the IN-REVIEW claim: `FCompileStateSummary` lives in `NiagaraDumpBuilder.h:17-22`, helpers at `NiagaraDumpBuilder.cpp:1147-1155` delegate to a template that branches on `IsNiagaraOnDemandCompileEnabled()` then `HasAnyUninitializedCompileStatus()`, `AttachNiagaraCompileStateToMeta` at `AssetDumpHandler.cpp:400-410` writes both fields only when `!Summary.bAvailable` (additive), `BuildAllFilesForAsset` at `AssetDumpHandler.cpp:799-806` dispatches for both `UNiagaraSystem` and `UNiagaraEmitter`, and `Utils/AssetDumpBuilder.cpp:146` bumps `dumpSchemaVersion` to 6 with a matching changelog comment. Needs a live `asset.dump` re-verify once the editor is back up.
- `#4-verify-fix` `DONE` tester — Verified: live `asset.dump` on `/App/App/LevelBlueprints/FX_Popping_the_Balloon.FX_Popping_the_Balloon` regenerated `meta.json` with top-level `compileStateAvailable: false` + `compileStateReason: "uninitialized"` and `dumpSchemaVersion: 6`, while `niagara_compile.json` still carries the `COMPILE_STATE_UNINITIALIZED` info issue — the additive promotion matches the IN-REVIEW contract.
