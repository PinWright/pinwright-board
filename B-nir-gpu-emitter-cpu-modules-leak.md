---
id: B-nir-gpu-emitter-cpu-modules-leak
title: "GPU emitters show CPU-only module stacks in NIR without flagging incompatibility"
status: WONTFIX
severity: Medium
category: bug
tags: [niagara, nir, gpu]
---

# GPU emitters show CPU-only module stacks in NIR

Emitters with `simTarget: GPUComputeSim` still emit full CPU-style parameter map structures in NIR, including modules that are not GPU-compatible (e.g. `SampleSkeletalMesh`, certain `SetVariables` patterns, mesh-sampling NDIs).

Result: NIR consumers see GPU emitter content that won't actually compile/execute on GPU, with no indication of the incompatibility.

## Sample paths

- `Game/Characters/.../NS_CharacterPortalDissolve/nir.txt` — Tunnel emitter at line 30 declares `simTarget: GPUComputeSim` but its Spawn/Update/ParticleSpawn stacks include CPU-only modules (`SetVariables`, `SpawnRate`, `SampleSkeletalMesh`)
- 10+ similar GPU emitters: `NS_CharacterSpawnIn`, `NE_BoneBasedEmission`, etc.

## Fix sketch

Two options:
1. **NIR filtering**: in `NIRDecompiler.cpp`, when emitter is GPU-sim, filter out modules not marked GPU-compatible. Emit a comment marker (`# excluded N CPU-only modules`).
2. **Compile-diagnostic flag**: don't filter, but add `niagara_compile.json` issues with `code: "GPU_INCOMPATIBLE_MODULE"` for each problematic module, plus an inline comment in NIR (`# GPU-incompatible: SampleSkeletalMesh`).

Option 2 is more faithful — the JSON shows what's actually authored, the comment surfaces the problem.

## Implementation notes

Implemented with the faithful diagnostic option.

Static review boundary:
- `Private/Handlers/Niagara/NiagaraDecompileHelpers.h` and `Private/Handlers/Niagara/NiagaraDumpBuilder.cpp` collect GPU-incompatible authored modules from `UNiagaraNodeFunctionCall` signature data.
- `Private/Handlers/Niagara/NiagaraDumpBuilder.cpp` appends `niagara_compile.json` issues with `code: "GPU_INCOMPATIBLE_MODULE"`.
- `Private/NIR/NIRDecompiler.cpp` keeps authored stack rows visible and annotates GPU-incompatible module rows inline with `GPU_INCOMPATIBLE_MODULE`.
- `Private/Tests/Niagara/TestNIRDecompiler.cpp` and `Private/Tests/Assets/TestNiagaraDumpBuilder.cpp` assert the NIR annotation and compile issue code.
- No Unreal build or runtime test has been run for this board transition.

## History
- `#1-gpu-emitter-cpu-leak` `OPEN` reporter — affects ~10 GPU emitters. NIR appears authoritative but masks incompatibility.
- `#2-skip-binary-not-rebuilt` `SKIP` tester — Fresh `asset.dump` of `/Game/Effects/Particles/Environmental/NS_CharacterPortalDissolve` shows 0 `GPU_INCOMPATIBLE_MODULE` markers in `nir.txt` and 0 issues with that code in `niagara_compile.json`, while the GPU emitter (`simTarget: GPUComputeSim` at line 74) still emits `SampleSkeletalMesh`, `SpawnRate`, `SetVariables` rows unannotated. Source code in `NiagaraDumpBuilder.cpp` / `NIRDecompiler.cpp` contains the fix, but the IN-REVIEW note states no rebuild has been performed, so the running editor binary is pre-fix. Cannot exercise the fix end-to-end without a rebuild.
- `#3-skip-still-not-rebuilt` `SKIP` tester — Re-ran `asset.dump` on `/Game/Effects/Particles/Environmental/NS_CharacterPortalDissolve`. `nir.txt` line 128 still shows `simTarget: GPUComputeSim` with `module SampleSkeletalMesh@v1.1`, `module SpawnRate`, and `module SetVariables_*` rows at lines 216–302 emitted with no `GPU_INCOMPATIBLE_MODULE` annotation; `niagara_compile.json` contains only a single `COMPILE_STATE_UNINITIALIZED` info issue and zero `GPU_INCOMPATIBLE_MODULE` codes. Binary is still pre-fix — same blocker as `#2`.
- `#4-returned-fix-not-emitting` `OPEN` tester — User confirmed the plugin binary HAS been rebuilt with the fix; re-verified end-to-end. Ran `asset.dump` on `/Game/Effects/Particles/Environmental/NS_CharacterPortalDissolve`; freshly written `nir.txt` still shows the GPU emitter `NE_BoneBasedEmission` at line 128 with `simTarget: GPUComputeSim` and CPU-only `module SetVariables_A433CC3844DFFF03D8CC4E8C2B90CBC4`, `module SpawnRate`, `module SampleSkeletalMesh@v1.1` rows in the `EmitterSpawn` stack at lines 216–302 with zero `GPU_INCOMPATIBLE_MODULE` annotations across the whole file. `niagara_compile.json` contains only `COMPILE_STATE_UNINITIALIZED` and zero `GPU_INCOMPATIBLE_MODULE` issues. Expected per the fix: GPU-incompatible authored module rows annotated inline with `GPU_INCOMPATIBLE_MODULE` and matching `code: "GPU_INCOMPATIBLE_MODULE"` issues in `niagara_compile.json`. Test: `asset.dump assetPath=/Game/Effects/Particles/Environmental/NS_CharacterPortalDissolve` then grep `GPU_INCOMPATIBLE_MODULE` across the dump folder (0 matches).
- `#5-not-a-bug-cpu-by-design` `WONTFIX` developer — Investigated against the actual asset + UE 5.7 Niagara source; the premise is invalid, there is nothing to flag. (1) Every module the reporter cited as a "CPU-only leak" (`SetVariables`, `SpawnRate`, `SampleSkeletalMesh` at nir.txt 216–302) is in the **EmitterSpawn** stack. Emitter and system scripts always execute on CPU even on a GPU emitter — only `ParticleSpawnScript..ParticleGPUComputeScript` are GPU scripts (`NiagaraScript.h:1032`; the engine's own `UNiagaraValidationRule_ModuleSimTargetRestriction` states "system and emitter scripts are always cpu scripts", `NiagaraValidationRules.cpp:1655`). So those CPU rows are correct, not a leak. (2) `SampleSkeletalMesh` also appears in the real ParticleSpawn/ParticleUpdate stacks (nir.txt 1581, 3420) and is GPU-capable: `UNiagaraDataInterfaceSkeletalMesh::CanExecuteOnTarget(...) const override { return true; }` (`NiagaraDataInterfaceSkeletalMesh.h:801`), overriding the base default of `false`. That is precisely why this asset legitimately ships as `GPUComputeSim`. (3) `niagara_compile.json` carries zero GPU errors. This explains why both prior signature-based attempts emitted nothing: there was genuinely nothing incompatible to emit. Any `GPU_INCOMPATIBLE_MODULE` annotation would be an active regression — false positives on CPU-by-design emitter/system modules and on GPU-capable DIs. A faithful static GPU-incompatibility classifier is not achievable in the decompiler; genuine GPU-incompatibility, when present, is reported by the Niagara compiler and already surfaces through `niagara_compile.json`. The real (separate) ergonomic gap behind the reporter's confusion — NIR not labelling which stacks run on CPU vs GPU — was discussed and deliberately not opened as a ticket. Closing as WONTFIX per user decision.
