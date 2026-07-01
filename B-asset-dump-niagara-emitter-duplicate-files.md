---
id: B-asset-dump-niagara-emitter-duplicate-files
title: "asset.dump writes byte-identical niagara_system.json and niagara_emitters.json for standalone NiagaraEmitter assets"
status: DONE
severity: Low
category: bug
tags: [asset-dump, niagara, duplicate-files]
---

# Standalone NiagaraEmitter dumps write the same content to two filenames

For all 44 `NiagaraEmitter` standalone assets in the cache, the dump
writes `niagara_system.json` and `niagara_emitters.json` with
**byte-identical** content — both contain the single emitter model
under `assetKind: NiagaraEmitter`.

The system-vs-emitter file naming scheme makes sense for `NiagaraSystem`
dumps (system-level identity vs. per-emitter data are genuinely
different aspects), but for a standalone emitter the same data ends
up in both files. Wastes disk and confuses readers who expect
`niagara_system.json` to mean "system identity / handles" and
`niagara_emitters.json` to mean "emitter array."

**Repro:** Inspect any standalone NiagaraEmitter dump, e.g.
`Game\Effects\Particles\Environmental\Emitters\NE_BoneBasedEmission\`.
The two files have identical SHA-256 hashes.

**Fix (proposed):** In the NiagaraEmitter dump branch, write only
`niagara_emitters.json` (or only `niagara_system.json`, whichever the
schema treats as canonical for emitter assets). Skip the duplicate
write entirely. Document the convention in the asset.dump wiki page so
readers know which file to expect for which `assetKind`.

## History
- `#1-initial-audit` `OPEN` reporter — 44 standalone NiagaraEmitter dumps each contain `niagara_system.json` and `niagara_emitters.json` with byte-identical content. Repro on `NE_BoneBasedEmission`.
- `#2-drop-duplicate-system-write` `IN-REVIEW` developer — Removed the `AddJsonFile(DumpFileNames::NiagaraSystem, ...)` line from the `UNiagaraEmitter` branch of `BuildAllFilesForAsset` in `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetDumpHandler.cpp`; standalone emitters now emit only `niagara_emitters.json` (the canonical emitter form), while the `UNiagaraSystem` branch is untouched and continues to write both files with genuinely different content. Added regression test `FAssetDumpStandaloneEmitterNoSystemFileTest` in `Source/EditorAutomationRpcGatewayTests/Private/Assets/TestNiagaraDumpBuilder.cpp` that drives `AssetDumpHandler::DumpSingleAsset` against a transient standalone `UNiagaraEmitter` and asserts `niagara_system.json` is absent while `niagara_emitters.json` is present.
- `#3-verify-fix` `DONE` tester — Verified: re-dumped `NE_BoneBasedEmission` (standalone NiagaraEmitter) — output folder now contains 8 files including `niagara_emitters.json` with `niagara_system.json` absent. Cross-check on `NiagaraShore_System` (standalone NiagaraSystem) still produces both files (9 total). Fix confirmed.
