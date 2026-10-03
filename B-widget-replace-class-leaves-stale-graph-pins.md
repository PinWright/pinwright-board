---
id: B-widget-replace-class-leaves-stale-graph-pins
title: "widget.replace_class leaves K2 variable nodes typed to the old widget class; the following compile doesn't reconstruct them and reports UpToDate"
status: OPEN
severity: Medium
category: bug
tags: [widget, widget.replace_class, blueprint-graph, variableget, stale-pin-type, silent-stale-data, dependency]
encounters: 1
lastSeen: 2026-09-30T00:00:00Z
rice: [1, 3, 1, 2]
priority: 17
---

# widget.replace_class leaves K2 variable nodes typed to the old widget class

`widget.replace_class` swaps the widget template and keeps its name (`WidgetAuthoringUtils.cpp:1195-1373`), then the handler only calls `MarkBlueprintAsStructurallyModified` (`WidgetReplaceClassHandler.cpp:87`). Nothing touches the event graph, so every `K2Node_VariableGet`/`VariableSet` for that widget keeps its output pin's `PinSubCategoryObject` on the old class. The saved package still imports the old class and its module (repro: `CommonTextBlock` and `/Script/CommonUI` stayed in W_LoadingScreenReasonDebugText after a swap to `TextBlock`), so a "drop this dependency" refactor fails without any error.

The later `blueprint.compile` reporting `UpToDate` is real, not skipped work. `CompileBlueprintWithDiagnostics` (`BlueprintHandlerUtils.cpp:465-516`) runs `FKismetEditorUtilities::CompileBlueprint` and reads `Blueprint->Status`. A non-load engine compile doesn't reconstruct nodes: STAGE IX in `BlueprintCompilationManager.cpp` calls `ReconstructAllNodes` only when `bIsRegeneratingOnLoad` is set, and `OptionallyRefreshNodes` otherwise acts only on hot reload. A stale subtype that is a subclass of the new property type still links, so the compile reports no error. When a later compile after `reconstruct_node` hits LIVE_INSTANCES_WOULD_BE_REINSTANCED, that is the intended guard (see `E-reinstancing-refusal-advises-closing-shared-map`).

The engine's own UMG "Replace With" (`WidgetBlueprintOperationUtils.cpp:1190-1219`) runs `FBlueprintEditorUtils::ReplaceVariableReferences` after the structural mark, "since the type might have changed". `replace_class` has no equivalent step.

**Workaround:** After the swap, run `blueprint.graph.find_nodes` for the widget's variable name, call `blueprint.graph.reconstruct_node` on each hit, then `blueprint.compile` (add `allowReinstancing` if live instances exist) and `asset.save`.

**Fix:** In `WidgetReplaceClassHandler.cpp`, after `MarkBlueprintAsStructurallyModified`, where the skeleton already carries the new property type:
- Reconstruct every `UK2Node_Variable` in the blueprint and its dependents whose `GetVarName()` equals `TargetName`, and also call `ReplaceVariableReferences(BP, TargetName, TargetName)` to match the engine.
- Return `reconstructedNodes: N` in the result.

Pin it with a test: swap a `CommonTextBlock` or `Button` subclass that has a graph VariableGet to its base class, then assert that the pin subtype is the new class.

## History
- `#1-stale-variableget-pin-type` `OPEN` reporter — `widget.replace_class {"targetName":"ReasonTextWidget","newType":"TextBlock","preserveProperties":true}` on W_LoadingScreenReasonDebugText returned `requiresCompile:true, oldClass:"CommonTextBlock"`. `blueprint.compile` then returned `UpToDate` with no errors or warnings, but the saved .uasset still contained `CommonTextBlock` and `/Script/CommonUI`. `blueprint.graph.find_nodes` matched `K2Node_VariableGet_0` on `pinSubTypeObjectName`/`pinSubTypeObjectPath`, and `blueprint.graph.reconstruct_node` fixed it. Checked against source: the handler never touches graph nodes, and the engine's non-load compile path never reconstructs them, so UpToDate is truthful and the stale state is silent. Distinct from DONE `F-widget-replace-class-preserve-children`, which covers the widget tree only.
