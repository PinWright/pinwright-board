---
id: B-niagara-decompile-model
title: "Niagara dumps need a compact decompile-only semantic model"
status: DONE
severity: High
category: feature
tags: [niagara, asset-dump, decompile, llm]
---

# Niagara dumps need a compact decompile-only semantic model

Niagara `asset.dump` output currently exposes detailed aspect files, including `niagara_graphs.json`. Those files are useful for raw inspection, but they are too large and too low-level as the first artifact for ordinary analysis.

## Expected

Add `niagara_model.json` and `niagara.decompile_model` as a compact, semantic-first model for `UNiagaraSystem` and `UNiagaraEmitter` assets.

The model should summarize systems, emitters, stack modules, parameters, renderers, script provenance, graph/node references, diagnostics, and structured reference records. Callers should read it before raw graph files when they need LLM-facing analysis.

Structured reference records should include reason, owner, class, object path, refs, properties, and provenance so unsupported or partially lowered Niagara features are still traceable.

## Scope Boundary

This is decompile-only for v1. Do not add compile/import, patching, or round-trip asset recreation behavior as part of this issue.

Raw `niagara_graphs.json` remains the pin-level reference artifact. The compact model links back to raw graph/script/property data instead of embedding full graph arrays.

## History

- `#1-niagara-model-plan` `OPEN` planner — Added a wave plan for `editor-automation.niagara-model.v1`, `niagara_model.json`, `niagara.decompile_model`, structured reference records, and explicit compile/import exclusion.
- `#2-niagara-model-implementation` `IN-REVIEW` developer — Added model builder, semantic lowering, dump/RPC wiring, wiki contract docs, and this board entry for review.
- `#3-niagara-model-review-fixes` `IN-REVIEW` developer — Fixed review findings: canonical dump filename reuse, renderer and advanced emitter feature coverage in system emitter entries, output-node-scoped stack lowering, module-scoped custom HLSL, depth-limit references for dynamic inputs, nested warning extraction, and emitter-model provenance cleanup.
- `#4-niagara-model-verify` `DONE` tester — Verified on `/App/App/FXE_Trail.FXE_Trail`: `niagara.decompile_model` RPC returns the full semantic model (assetKind, emitters with stackModules/renderers/advancedFeatures/parameters, diagnostics) and `asset.dump` writes `niagara_model.json` to the dump directory. Reference records carry reason/owner/refs/provenance fields (e.g. `renderer_material`, `dynamic_input_not_semantically_lowered`) linking back to raw graph files as designed.
