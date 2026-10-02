---
id: F-niagara-get-compiled-script
title: "No verb exposes a Niagara script's compiled output (generated HLSL, VM assembly, op count, GPU permutations)"
status: IN-REVIEW
severity: Medium
category: feature
tags: [niagara, compile, hlsl, gpu, diagnostics, gap-analysis-2026-09-30]
---

# Compiled Niagara script output is unreadable

PinWright reports compile status, errors, `hasGpuSimulation` and the GPU compute script's graph
(`Handlers/Niagara/NiagaraDumpBuilder.cpp:1386`, `niagara.compile_status`). NIR emits that
script's graph, not its HLSL. No source reads `LastHlslTranslation*`,
`LastAssemblyTranslation`, `LastOpCount` or the GPU shader script. An agent debugging a custom
HLSL compile error or GPU cost cannot see what the translator produced. The editor shows this
on its Generated Code tab, which simply reads the fields below
(`SNiagaraGeneratedCodeView.cpp:426-460`).

## Competitors (compare row "Compiled GPU script inspection", 2026-09-30)
- **Monolith: real but fragile.** `get_compiled_gpu_hlsl`, `MonolithNiagaraActions.cpp:3125`,
  impl `:7567-7615`.
  - GPU only; it errors on CPU emitters (`:7581`).
  - If both HLSL fields are empty it does a NON-forced `RequestCompile(false)` + wait, which is
    skipped on a compile-id match or DDC hit (`NiagaraSystemCompilingManager.cpp:390-391`).
  - Returns the raw text of `LastHlslTranslationGPU`, falling back to `LastHlslTranslation`,
    then `LastAssemblyTranslation`. No truncation, no stats.
- **ue-mcp: thin.** `GetCompiledHLSL`, `NiagaraHandlers.cpp:1844-1876`, returns a name and
  `IsCompilable()`, which answers "can compile", not "has compiled". No text.
- **Epic:** compile state and events only (`GetSystemCompileState`).

## UE APIs (5.8)
`FNiagaraVMExecutableData` is at `NiagaraScript.h:399`.
- **Always present:** `ByteCode` :407, `NumTempRegisters` :411, `CompileTags` :440, `Attributes`
  :447, `SimulationStageMetaData` :514, `LastCompileStatus` :511.
- **WITH_EDITORONLY_DATA:** `Parameters` :420, `DataInterfaceInfo` :468, `ErrorMsg` :529,
  `LastCompileEvents` :533.
  - `LastHlslTranslation` :487 — **Transient**.
  - `LastHlslTranslationGPU` :491 — persisted.
  - `LastAssemblyTranslation` :494 — **Transient**.
  - `LastOpCount` :497 — **Transient**.
- **Accessors:** `UNiagaraScript::GetVMExecutableData()` :1284; GPU script via
  `FVersionedNiagaraEmitterData::GetGPUComputeScript()` (`NiagaraEmitter.h:425`);
  `GetRenderThreadScript()` :1132 → `FNiagaraShaderScript` (`NiagaraShared.h`):
  `GetNumPermutations` :888, `IsCompilationFinished` :779, `GetCompileErrors` :783.

**Drift and pitfalls**
- **Transient fields.** CPU HLSL, assembly and op count are Transient on 5.4+ (not on 5.3,
  `UE_5.3 NiagaraScript.h:513`). They are empty after an editor restart or any DDC-hit compile.
  GPU HLSL usually survives a load.
- **Removed fields.** `OptimizedByteCode` was removed in 5.7. There is no per-script compile time
  in the executable data.
- **Forcing a compile dirties the package.** Only `RequestCompile(true)` really re-translates. It
  calls `ForceGraphToRecompileOnNextCheck`, which assigns a new ForceRebuildId and calls
  `Modify()` on graphs (`NiagaraSystem.cpp:3814-3816`, `NiagaraGraph.cpp:3847`). That dirties the
  package.
- **No keep-HLSL switch.** No cvar retains HLSL. `fx.ForceNiagaraTranslatorDump` only dumps to
  disk (`NiagaraCompiler.cpp:63`).

## Proposed verb

`niagara.get_compiled_script` is a read verb.

**Params**
- `assetPath` (required).
- `emitter?`, `scriptUsage?`: filters.
- `include`: default `["stats"]`; add `"hlsl"` and/or `"assembly"` to opt in.
- `maxChars`: default 20000.
- `forceCompile`: default false.

**Behavior**
- Walk the system spawn/update scripts plus `EmitterData->GetScripts(Out, true)` and the GPU
  compute script.
- Per script return `{usage, simTarget, compileStatus, hlsl | hlslMissing:{reason, hint},
  truncated, totalChars, assembly?, opCount, byteCodeBytes, numTempRegisters, attributeCount,
  dataInterfaceCount, simStages, gpu?:{permutations, shaderCompileFinished, shaderErrors},
  compileEvents}`.
- Missing text is reported as `hlslMissing` with the reason (transient field, not compiled in
  this session), never as an empty string.
- `forceCompile: true` runs `RequestCompile(true)` then waits (depends on
  `B-niagara-compile-wait-does-not-wait`). It snapshots and restores every touched package's dirty
  flag, per the `rpc-design.md` checklist: a read verb leaves dirty flags exactly as it found them.
  The response reports `forcedCompile: true`.
- Large text follows the plugin's existing oversize-spill convention (check which applies
  before implementing).

## Acceptance
- A GPU emitter returns non-empty `hlsl` plus a permutation count.
- A CPU script right after an editor restart returns `hlslMissing` with a reason, not `""`.
- With `forceCompile: true` the same CPU script returns `hlsl`, and the package dirty flag equals
  its value before the call.
- `maxChars` truncation reports `truncated: true` and `totalChars`.
- `include` omitted returns stats only.

Effort S-M. Low risk: read-only, all fields exist on 5.3+, and the Transient drift is reported
rather than hidden. Pairs with `F-niagara-create-module-script` for debugging custom HLSL.

## History
- `#1-filed-from-gap-analysis` `OPEN` reporter — Filed from the 2026-09-30 competitor gap analysis (compare row "Compiled GPU script inspection": PinWright partial, Monolith yes). Monolith's implementation was read and verified at `:7581-7610`; ue-mcp is thin. Engine citations are UE 5.8 source reads with the 5.3/5.7 drift checked. Nothing was built or run.
- `#2-implemented` `IN-REVIEW` developer — New read verb `niagara.get_compiled_script` in `Handlers/Niagara/NiagaraGetCompiledScriptHandler.cpp` (+ shared `NiagaraScriptCompileReport.h/.cpp`). It walks system spawn/update plus each handle's `GetScripts(.., true)` (GPU compute included), or a standalone emitter's or script's scripts, with `emitter` / `scriptUsage` filters; an empty match is `TARGET_NOT_FOUND`. `include` defaults to `["stats"]`: byteCodeBytes, numTempRegisters, attributeCount, dataInterfaceCount, simStages, opCount (null unless the assembly is retained), compileEvents, and for GPU `gpu{permutations, shaderCompileFinished, shaderCompileSucceeded, shaderErrors}` via exported `IsScriptCompilationPending` / `DidScriptCompilationSucceed`, so no NiagaraShader link is needed. `hlsl` / `assembly` honour `maxChars` with `<field>Truncated` / `<field>TotalChars`, and missing text is `<field>Missing{reason,hint}`, never "". `forceCompile` on systems uses the bounded wait (`RequestNiagaraCompile` + `WaitForSystemCompile`) with the rapid-iteration snapshot/merge from niagara.compile; on scripts it uses the synchronous `RequestCompile`. Either way the package dirty flag is restored only when the compile dirtied a clean package (preserve, not clear). Standalone emitters refuse `forceCompile` (INVALID_ARGUMENT) because their scripts compile only inside systems. Large text relies on the transport's existing oversize spill. Tests: `PinWright.niagara.get_compiled_script.{StatsByDefaultTextOnRequest,ForceCompilePreservesDirtyFlag,GpuEmitterReportsHlslAndPermutations}` (`Tests/Niagara/TestNiagaraGetCompiledScript.cpp`). Wiki: `niagara.md`. Compile-checked with UBT -SingleFile; not yet run.
- `#3-fixround-selector` `IN-REVIEW` developer — Full-suite fix round. `StatsByDefaultTextOnRequest` and `ForceCompilePreservesDirtyFlag` failed "exactly one script matched: 3". The verb was right and the tests were wrong: the stock SimpleExplosion fixture has three emitters, each with a ParticleUpdate script, so a usage-only filter correctly returns three rows. Both tests now pass `emitter` (the fixture's first handle) beside `scriptUsage`. `GpuEmitterReportsHlslAndPermutations` passed in that run (non-empty GPU HLSL and permutation count measured). Compile-checked with -SingleFile; not re-run.
