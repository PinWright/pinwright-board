---
id: F-niagara-decompile-nir-overrides
title: "NIR v1b — module input override-chain resolver"
status: DONE
severity: High
category: feature
tags: [niagara, ir, decompile, nir]
---

# NIR v1b — module input override-chain resolver

Replace v1a's `# TODO: dynamic-input chain (F-niagara-decompile-nir-overrides)` placeholders with real `input X = <expr>` lines. Walk the `UNiagaraNodeParameterMapSet` override node returned by `NiagaraResetModuleInput::FindStackFunctionOverrideNode` (declared in NiagaraResetModuleInputHelpers.h). For each input pin's upstream chain, classify and emit:

- direct literal → scalar / struct / vector value
- `UNiagaraNodeInput` → `$Scope.Name` linked parameter
- chain of `UNiagaraNodeFunctionCall` → `dynamic ModuleName { input ... = ... }` (recursive)
- static-switch override → `static X = ...`

Dynamic-input is the recursive case — implement as one mutually-recursive `EmitInputValueExpr(UEdGraphPin*)` function. Module version pinning attaches as `module ScriptName@vMajor.Minor` when `SelectedScriptVersion != FGuid()` (already in v1a; v1b confirms it during expression emission).

## Files (v1b)
- Modified: `Private/NIR/NIRDecompiler.cpp` only.

## Depends on
- `F-niagara-decompile-nir` (v1a) must have landed grammar slots and the locked placeholder string.

## Effort
~700 LoC, 3–4 dev-days. No new files.

## History
- `#1-initial-scope` `OPEN` reporter — Spun off from F-niagara-decompile-nir during v1a sprint. The override-chain resolver is complication #1 from the original ticket — the "hard problem" — separated so v1a can ship the grammar publicly first.
- `#2-implementation` `IN-REVIEW` developer 2026-05-14 — Implemented `EmitInputValueExpr` in `Private/NIR/NIRTextEmitter.{h,cpp}` (declaration was wave-1 carved; wave-2 fills it in). Classifies override-pin upstream chains into literal / `$Scope.Name` linked-param / `dynamic ModuleName { ... }` recursive dynamic-input / static-switch (decoded separately via `BuildStaticSwitchInputs`). Depth guard at 32 emits `# recursion-limit-reached` plus an `FNIRResult.Warnings` entry. `NIRDecompiler.cpp`'s `AppendOverrideInputs` now collects override-node pins by un-aliased input name and routes each through `EmitInputValueExpr`; the v1a `# TODO: dynamic-input chain` placeholder is removed everywhere (source + wiki). Tests in `TestNIRDecompiler.cpp` cover literal-float, linked-param, dynamic-input, static-switch coexistence, and recursion-depth via the SVM-backed fixtures in `TestNIRFixtures`. Files touched: `Private/NIR/NIRTextEmitter.cpp`, `Private/NIR/NIRDecompiler.cpp`, `Private/Tests/Niagara/TestNIRDecompiler.cpp`, `docs/wiki/niagara.md`.
- `#3-verify-live-nir-overrides` `DONE` tester — Verified: `niagara.decompile_nir` on `/Game/Vefects/Free_Fire/Shared/Particles/NS_Fire_Big` returned 252 `input` lines including concrete enum/bool override expressions and no `# TODO: dynamic-input chain`; `/Game/UltraDynamicSky/Particles/Rain` returned 448 `input` lines and also no placeholder. Earlier object-path retry on `/Niagara/DefaultAssets/Templates/Systems/SimpleExplosion.SimpleExplosion` returned `INVALID_PARAMS`, while the package path `/Niagara/DefaultAssets/Templates/Systems/SimpleExplosion` decompiled successfully with no placeholder.
