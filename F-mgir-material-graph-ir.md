---
id: F-mgir-material-graph-ir
title: "MGIR bulk material graph authoring"
status: DONE
severity: High
category: feature
tags: [mgir, material, graph, ir, rpc]
---

# MGIR Bulk Material Graph Authoring

Add MGIR (Material Graph IR), a text-based bulk authoring and decompilation path for material and material-function assets. The feature provides `material.compile_mgir` and `material.decompile_mgir`, shares generic IrCore tokenizer/type parsing pieces with BPIR where safe, and routes material expression creation through a shared reflected-property factory.

The feature is additive. Existing `material.authoring.*`, `material.graph.*`, and `blueprint.compile_bpir` APIs remain available.

**Fix:** Implement IrCore foundations, MGIR grammar/parser/emitter/pin resolver/decompiler/layout/compiler, material expression factory support, RPC handlers, wiki docs, and focused tests. Preserve v1 composite behavior as decompile-only with `MGIR_COMPOSITE_NOT_SUPPORTED` on compile.

## History
- `#1-plan-refined` `OPEN` reporter — Refined the old home-scoped MGIR plan against current plugin code. Current namespace is `material.authoring.*`, `material.graph.add_expression` already exists but needed `properties`, BPIR remains BPIR-named, and current wave-worker review output is inline rather than sidecar-based.
- `#2-implemented-mgir` `IN-REVIEW` developer — Implemented MGIR through wave-worker chunks. Added IrCore tokenizer/type foundations, `FMaterialExpressionFactory`, `material.graph.add_expression` `properties`, material-authoring shims, MGIR parser/emitter/resolver/decompiler/layout/compiler, `material.compile_mgir` and `material.decompile_mgir` handlers, material-function compile support, docs/wiki overlay, and focused material handler tests. Static checks found no conflict markers, stale `material_authoring.*`, `FIrTypeSpecParser`, or `MGIR_FUNCTION_COMPILE_NOT_SUPPORTED`; build/tests were not run.
- `#3-verified-mgir-roundtrip` `DONE` tester — Verified: `material.decompile_mgir` returned MGIR text and no warnings for `/Game/UI/Foundation/Materials/M_UI_Throbber_Base`; created `/Game/App/UI/Test/M_McpReviewTemp_20260429`, decompiled it, then `material.compile_mgir mode:"append" save:false` compiled one block with `expressionsCreated:1` and returned the temp material path.
