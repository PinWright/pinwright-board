---
id: B-niagara-dump-live-state-noise
title: "Niagara authored dump sidecars include live compile-session state"
status: OPEN
severity: Medium
category: bug
tags: [asset-dump, niagara, determinism, compile-state, nir]
---

# Niagara authored dump sidecars include live compile-session state

`asset.dump` writes source-controlled `niagara_compile.json` and `nir.txt` from live editor state, so compiling an unchanged Niagara asset creates false authored diffs. A fresh `/App` sweep changed `FXE_Trail`: `niagara_compile.json` flipped `hasOutstandingCompilationRequests`, `valid`, and four `compileStatus` values, while `nir.txt` flipped `needsRecompile` and lost two compile-populated `InitializeParticle.Lifetime` rapid-iteration rows. No source asset edit was involved.

The pre-fix code wires `BuildCompileJson` directly into the default dump (`AssetDumpHandler.cpp:536-553`); that builder reads `GetLastCompileStatus`, `IsValid`, outstanding/active compilation, readiness, and the global on-demand-compile CVar (`NiagaraDumpBuilder.cpp:198-222,1620-1705`). NIR likewise reads raw `RapidIterationParameters`, emits the global `# compile-state-stale` annotation, and serializes `Handle.IsValid()` / `Handle.NeedsRecompile()` (`NIRDecompiler.cpp:180-201,571-585,688-692`). These are valid diagnostics but not stable authored state.

This does not reopen `B-asset-dump-niagara-compile-state-stale[-followup]`: those DONE tickets made uninitialized live state honest. It also excludes Niagara compiled fields in generic `properties.json`, tracked by `B-asset-dump-derived-properties`. This ticket separates live diagnostics from the two authored sidecars.

**Workaround:** ignore live compile fields and compile-generated rapid-iteration rows when reviewing Niagara dump diffs, or compare only after reproducing the same compile session state.
**Fix:** keep live compile/readiness data on `niagara.inspect` and `niagara.validate`, but make default `niagara_compile.json` session-independent; remove NIR handle/CVar diagnostics; derive rapid-iteration rows from stable authored override provenance rather than blindly dumping the raw compiled parameter store. Preserve genuine authored rapid-iteration overrides. Bump both `niagara_compile.json` and `nir.txt` aspect versions.

Acceptance tests must dump the same fixture before and after Niagara compilation and assert byte-identical authored sidecars, while proving diagnostic RPC output can still change. A fixture must also retain one authored rapid-iteration override and exclude a compile-generated/default entry.

## History
- `#1-app-redump-live-state` `OPEN` reporter — `asset.dump_folder {folderPath:"/App",recursive:true}` changed only live Niagara session state for `/App/App/FXE_Trail`: compile readiness/status fields flipped, `needsRecompile` changed true→false, and two `InitializeParticle.Lifetime` rapid rows disappeared after compilation. Source inspection confirms the default dump reads those live surfaces directly.
