---
id: F-niagara-decompile-nir-script-graphs
title: "NIR v1c — module-script graph emission"
status: DONE
severity: Medium
category: feature
tags: [niagara, ir, decompile, nir]
---

# NIR v1c — module-script graph emission

Implement the `script "/Path" { graph <Usage> { ... } }` scope. Walks `UNiagaraGraph` nodes for each script-usage entry point (ParticleSpawn/Update/SystemUpdate/Module/etc.). Emits one line per node — `UNiagaraNodeOp`, `UNiagaraNodeParameterMapGet/Set`, `UNiagaraNodeFunctionCall`, `UNiagaraNodeInput`, `UNiagaraNodeOutput`, `UNiagaraNodeStaticSwitch` — mirroring BPIR's node-by-node emission. Reuses v1b's `EmitInputValueExpr` for input wiring; pin names use `%local` form; positions use `@(x, y)`. Node opcode list comes from `UNiagaraNodeOp::OpName`.

Replaces v1a's `# script graph emission deferred (F-niagara-decompile-nir-script-graphs)` placeholder line.

## Files (v1c)
- Modified: `Private/NIR/NIRDecompiler.cpp` only.

## Depends on
- `F-niagara-decompile-nir` (v1a)
- `F-niagara-decompile-nir-overrides` (v1b) — shares `EmitInputValueExpr`

## Effort
~700 LoC, 3 dev-days.

## History
- `#1-initial-scope` `OPEN` reporter — Spun off from F-niagara-decompile-nir during v1a sprint. Module-script graph emission is the third nested scope in the original NIR grammar; deferred so v1a can ship system/emitter coverage first.
- `#2-implementation` `IN-REVIEW` developer 2026-05-14 — Replaced v1a's `# script graph emission deferred (F-niagara-decompile-nir-script-graphs)` placeholder with realised `graph <Usage> { ... }` body emission. Dispatcher (wave-3) walks every `UNiagaraNode` through three family emitters (`NIRGraphEmit_Dataflow` / `NIRGraphEmit_Control` / `NIRGraphEmit_Util`); unhandled classes fall through to `# unknown-node <ClassName>` plus an `FNIRResult.Warnings` entry. Wave-4 lands the three families across `Private/NIR/NIRGraphEmitter_Dataflow.cpp` (UNiagaraNodeOp / UNiagaraNodeParameterMapGet / UNiagaraNodeParameterMapSet / UNiagaraNodeFunctionCall / UNiagaraNodeInput / UNiagaraNodeOutput), `Private/NIR/NIRGraphEmitter_Control.cpp` (UNiagaraNodeStaticSwitch / UNiagaraNodeIf / UNiagaraNodeSelect / UNiagaraNodeUsageSelector / UNiagaraNodeSimTargetSelector), and `Private/NIR/NIRGraphEmitter_Util.cpp` (UNiagaraNodeCustomHlsl / UNiagaraNodeConvert / UNiagaraNodeReroute). Pin wiring reuses v1b's `EmitInputValueExpr` for input override expressions and the new `FormatPinValueRef` for SSA-style "%upstream.OutputPinName" inter-node refs. Per-family tests cover one case per node class (`TestNIRGraphDataflow.cpp` / `TestNIRGraphControl.cpp` / `TestNIRGraphUtil.cpp`); reroute caveat (logical pass-through, emitted for traceability) recorded in the wiki under `feedback_ir_logical_not_visual`. Files touched: `Private/NIR/NIRTextEmitter.{h,cpp}`, `Private/NIR/NIRDecompiler.cpp`, `Private/NIR/NIRGraphEmitter_Dataflow.cpp`, `Private/NIR/NIRGraphEmitter_Control.cpp`, `Private/NIR/NIRGraphEmitter_Util.cpp`, `Private/Tests/Niagara/TestNIRGraphDataflow.cpp`, `Private/Tests/Niagara/TestNIRGraphControl.cpp`, `Private/Tests/Niagara/TestNIRGraphUtil.cpp`, `docs/wiki/niagara.md`.
- `#3-verify-nir-graph` `DONE` tester — Verified: `asset.search_assets classNames:["NiagaraScript"]` found `/Game/Effects/NiagaraModules/NM_ImpactSpawnDataChannel.NM_ImpactSpawnDataChannel`, then `niagara.decompile_nir assetPath:"/Game/Effects/NiagaraModules/NM_ImpactSpawnDataChannel"` returned `script "/Game/Effects/NiagaraModules/NM_ImpactSpawnDataChannel.NM_ImpactSpawnDataChannel" { graph Module { ... } }` with concrete `output`, `input`, `get`, `set`, `call`, and `reroute` node lines and no `# script graph emission deferred (F-niagara-decompile-nir-script-graphs)` placeholder.
