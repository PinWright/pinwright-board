---
id: E-widget-add-misses-bisvariable
title: "widget.add registers GUID map entry but never sets bIsVariable=true"
status: DONE
severity: Medium
category: ergonomic
tags: [widget, bpir, variable-promotion]
---

# widget.add registers GUID map entry but never sets bIsVariable=true

`widget.add` calls `EnsureWidgetVariableGuid` (WidgetAddHandler.cpp:100), which
adds the name to `WidgetVariableNameToGuidMap` and fires `OnVariableAdded`, but
the handler never sets `Widget->bIsVariable = true`. This leaves a half-promoted
state:

- `WidgetXmlExporter.cpp:286` reports `IsVariable="true"` because it derives the
  attribute from `WidgetVariableNameToGuidMap` membership, not from the widget's
  `bIsVariable` flag.
- The blueprint recompile path keys variable property generation off
  `bIsVariable`, so no FObjectProperty is created and BPIR `$Name` resolution
  fails on the next `blueprint.compile_bpir`.

`widget.import_xml` does both — `WidgetXmlImportHandler.cpp:452` sets
`Widget->bIsVariable = true` before calling `EnsureWidgetVariableGuid`. That is
the pattern `widget.add` should match (referenced by DONE
`E-widget-var-promotion` which gave import_xml its `requiresCompile` flag).

**Workaround:** Delete the widget and re-add via `widget.import_xml` mode:"add",
which sets `bIsVariable=true` and returns `requiresCompile: true`.

**Fix:** In WidgetAddHandler.cpp, set `Widget->bIsVariable = true` before the
existing `EnsureWidgetVariableGuid(WidgetBP, Widget->GetFName())` call, and
return `requiresCompile: true` in the result JSON for parity with
`widget.import_xml`.

## History
- `#1-half-promotion-trap` `OPEN` reporter — Added `ReasonText` TextBlock to `/App/App/UI/W_PhotoPopup` via `widget.add`; `widget.export_xml` showed `IsVariable="true"` but `blueprint.compile_bpir` with `call SetText(Target: $ReasonText, ...)` failed `Unresolved function: 'SetText'`. Verified handler at `WidgetAddHandler.cpp:100` only calls `EnsureWidgetVariableGuid` — does not set `Widget->bIsVariable = true`. Compare `WidgetXmlImportHandler.cpp:452` which sets both. Exporter reads from GUID map (`WidgetXmlExporter.cpp:286`), hiding the inconsistency.
- `#2-set-bisvariable-and-requirescompile` `IN-REVIEW` developer — In `WidgetAddHandler.cpp` set `Widget->bIsVariable = true` before `EnsureWidgetVariableGuid` (line 107) and return `requiresCompile: true` in the result JSON, matching `widget.import_xml` behavior.
- `#3-verify-fix` `DONE` tester — Verified: created temp BP `/Game/App/UI/Test/W_McpVerifyTemp_E_widget_add`, called `widget.add` (type=TextBlock, name=ReasonText) → response included `requiresCompile: true`. After `blueprint.compile`, `blueprint.compile_bpir` with `call SetText(Target: $ReasonText, InText: ...)` resolved `$ReasonText` as a TextBlock-typed variable (failed only on unrelated FText literal syntax — no "Unresolved function: 'SetText'" symptom from the original repro). Temp BP deleted.
