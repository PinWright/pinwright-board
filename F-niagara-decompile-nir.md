---
id: F-niagara-decompile-nir
title: "Niagara decompile-only text IR (NIR) — v1a System+Emitter shell"
status: DONE
severity: High
category: feature
tags: [niagara, ir, decompile, text-format, nir]
---

# Niagara decompile-only text IR (NIR) — v1a System+Emitter shell

v1a scope: System+Emitter shell. The decompiler emits `system "..."` and standalone `emitter "..."` blocks with user parameters, per-emitter `simTarget`, renderers / simStages / eventHandlers via reflected-property dispatch, stack rows of the form `module ShortName[@vMajor.Minor] @<order> enabled|disabled` for `SystemSpawn`/`SystemUpdate`/`EmitterSpawn`/`EmitterUpdate`/`ParticleSpawn`/`ParticleUpdate`, and `static X = ...` lines for static-switch inputs. Override-chain module inputs render as `input X = # TODO: dynamic-input chain (F-niagara-decompile-nir-overrides)` placeholders preserving the input pin name so v1b substitutes the resolved expression in place. Standalone `UNiagaraScript` assets emit a single-line `# script graph emission deferred (F-niagara-decompile-nir-script-graphs)` placeholder under the `script "..." { ... }` header. When `fx.Niagara.OnDemandCompile=1` the decompiler prepends a `# compile-state-stale` annotation line and adds a matching warning string so consumers can distinguish post-load deferred state from a fresh compile.

Dual-surface delivery: the plugin's `IrSidecarRegistry` is the canonical dual-surface plumbing. `NIRDecompileHandler.cpp` calls `REGISTER_DECOMPILE_IR` three times (one each for `UNiagaraSystem`, `UNiagaraEmitter`, `UNiagaraScript`) — all three thunks forward to a single `BuildNIRSidecar` that calls `NIRDecompiler::BuildNiagaraIrText(UObject*)`. The same handler registers `niagara.decompile_nir` as an RPC method (also calling `BuildNiagaraIrText`). The asset-dump sidecar `nir.txt` and the RPC response body are byte-identical because both call the one shared builder; the dual-surface zero-divergence invariant is enforced by the registry and exercised by `FNiagaraNirRpcAndDumpParityTest`.

## Out of scope (deferred)

- **Override-chain resolution** — spun off to `F-niagara-decompile-nir-overrides` (v1b). v1a leaves the placeholder string above so v1b can substitute one expression at a time without rewriting the line shape.
- **Module-script graph emission** (`graph <Usage> { ... }` node-by-node body inside `script "..."`) — spun off to `F-niagara-decompile-nir-script-graphs` (v1c).
- **Bidirectional NIR** (`niagara.compile_nir`) — deferred to v2; sketch lives in this ticket's history #1 and the original "v2 (deferred)" section.

## Files touched (v1a)

New:
- `Source/PinWright/Private/NIR/NIRDecompiler.h`
- `Source/PinWright/Private/NIR/NIRDecompiler.cpp`
- `Source/PinWright/Private/Handlers/Niagara/NIRDecompileHandler.cpp`
- `Source/PinWright/Private/Tests/Niagara/TestNIRDecompiler.cpp`
- `docs/board/F-niagara-decompile-nir-overrides.md` (v1b spin-off)
- `docs/board/F-niagara-decompile-nir-script-graphs.md` (v1c spin-off)

Modified:
- `Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.h` — adds `DumpFileNames::Nir`.
- `Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.cpp` — adds `DumpFileNames::Nir` to the baseline `FixedCanonical[]` array.
- `docs/wiki/niagara.md` — replaces the "no text IR" caveat with an NIR section.

## Effort

~1,200 LoC, 4–5 dev-days for v1a. v1b and v1c add ~700 LoC each (each is a single-file extension of `NIRDecompiler.cpp`).

## See also

- `B-asset-dump-niagara-compile-state-stale` and its follow-up — compile-state honesty (complication #5) is implemented as the `# compile-state-stale` annotation + `IsNiagaraOnDemandCompileEnabled()` check.
- `B-niagara-static-switch-not-decoded` — static-switch decoding is shared via `NiagaraDumpBuilder::BuildStaticSwitchInputs`.
- `docs/wiki/niagara.md` — replaces the "no text IR" caveat.

## History
- `#1-initial-scope` `OPEN` reporter — Filing decompile-only NIR (v1).
  Verified in code that `NiagaraModelBuilder.cpp` and
  `NiagaraDumpBuilder.cpp` exist under
  `Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/` and
  cover ~60% of the system/emitter/stack traversal needed. Verified
  that no existing board ticket covers Niagara IR / NIR (greppped
  `docs/board/` for `niagara`, `nir`, `IR`). Verified that
  `docs/wiki/niagara.md:152-153` documents the "no text IR" gap this
  ticket closes. Grammar sketch and 6 Niagara-specific complications
  captured above; v2 (compile path) explicitly deferred. Soft
  prereqs: `F-ircore-reflected-property-emit`,
  `R-ir-grammar-harmonize`, `F-ir-authoring-guide`,
  `R-ircore-test-fixture`.
- `#2-dual-surface-mandate` `OPEN` reporter 2026-05-13 — Locking in the
  **dual-surface invariant** before implementation. BOTH surfaces are
  MANDATORY, neither is optional: (a) asset-dump sidecar `nir.txt`
  registered in `AssetDumpHandler.h` `DumpFileNames` and emitted from
  the Niagara System / Emitter / Module-Script dispatch branches in
  `AssetDumpHandler.cpp`, written under
  `.editor-automation/asset-dumps/.../<asset>/nir.txt`; (b) MCP RPC
  `niagara.decompile_nir` handler at
  `Private/Handlers/Niagara/NIRDecompileHandler.cpp`, callable via
  `call("niagara.decompile_nir", { assetPath })`, returns
  `{ ir, warnings }`. Both surfaces MUST share ONE builder function
  `BuildNiagaraIrText(UObject* NiagaraAsset) → FIrResult { Text, Warnings, bSuccess }`
  (dispatching internally on `UNiagaraSystem` vs `UNiagaraEmitter` vs
  `UNiagaraScript`) living in `Private/NIR/NIRDecompiler.cpp`. The
  sidecar pipeline calls it for dump emission; the RPC dispatch calls
  it for direct response. Zero divergence between the two — never ship
  one without the other, never let the two implementations drift.
  Compile-state honesty rules from complication #5 apply uniformly
  through the shared builder (single tag-stale path, not two). If the
  dump branch needs format differences, route them through builder
  options, not a parallel implementation.
- `#3-reformulated-and-v1a-implemented` `IN-REVIEW` developer — Reformulated to v1a (System+Emitter shell) after analysis confirmed IrSidecarRegistry already enforces the dual-surface invariant and 5 of 6 complications are solved in existing NiagaraDumpBuilder/NiagaraModelBuilder/NiagaraResetModuleInputHelpers code. Spun off two follow-up tickets: F-niagara-decompile-nir-overrides (v1b — override-chain resolver) and F-niagara-decompile-nir-script-graphs (v1c — module-script graph emission). Implemented: Private/NIR/{NIRDecompiler.h,NIRDecompiler.cpp}; Private/Handlers/Niagara/NIRDecompileHandler.cpp registering niagara.decompile_nir + three IrSidecarRegistry entries (System/Emitter/Script). nir.txt sidecar wired via AssetDumpHandler.h DumpFileNames::Nir + AssetDumpHandler.cpp FixedCanonical[]. Wiki docs/wiki/niagara.md updated. Tests in Private/Tests/Niagara/TestNIRDecompiler.cpp cover system/emitter/script-placeholder shape, RPC/sidecar parity, compile-state-stale annotation. Override-chain inputs emit `# TODO: dynamic-input chain (F-niagara-decompile-nir-overrides)` placeholders preserving pin names; v1b substitutes cleanly. Module-script branch emits `# script graph emission deferred (F-niagara-decompile-nir-script-graphs)`.
- `#4-verify-system-nir` `DONE` tester — Verified: `niagara.decompile_nir` on `/Game/Effects/Particles/Item/NS_Heal.NS_Heal` returned v1a NIR with `system`, emitter, renderer, stack module, static, and input rows; `asset.dump` on `/Game/Effects/Particles/Item/NS_Heal` wrote `.editor-automation/asset-dumps/Game/Effects/Particles/Item/NS_Heal/nir.txt`, whose opening system path and first stack/module rows matched the RPC output shape.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 6 body citations repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
