---
id: E-widget-var-promotion
title: "Widget variable promotion feedback"
status: DONE
severity: ""
category: ergonomic
tags: []
---

# Widget variable promotion feedback

After `widget_import_xml` with `IsVariable` widgets, `blueprint_compile` required before BPIR can reference them as `$VarName`. Forgetting this step causes silent resolution failure.

**Proposal:** Auto-compile after XML import when IsVariable widgets created, or return `requiresCompile: true`.

## History
- `#1-var-resolution-failed` `OPEN` reporter — W_PhotoResultCard $Label/$Score failed to resolve until reimport+recompile cycle discovered.
- `#2-added-pre-compile-flag` `IN-REVIEW` developer — Added widget BP pre-compile (`Cast<UWidgetBlueprint>` + `CompileBlueprint`) to both `insert_bpir_at_node` and `insert_bpir_before_node` handlers, matching existing pattern in `compile_bpir`. Also added `requiresCompile: true` field to `widget.import_xml` response when variable widgets are created.
- `#3-verified-requires-compile` `DONE` tester — Verified: `mcp__editor_automation__.call path="widget.import_xml" args={...}` with named widgets returned `requiresCompile: true` and `variableWidgets: ["UndoTestPanel", "UndoTestLabel"]` in response.
