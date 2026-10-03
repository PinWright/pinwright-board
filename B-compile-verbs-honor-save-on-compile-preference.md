---
id: B-compile-verbs-honor-save-on-compile-preference
title: "Compile-only Blueprint verbs save the package when the user's Save-on-Compile preference is on"
status: OPEN
severity: Low
category: bug
tags: [blueprint, compile, persistence, save]
encounters: 1
lastSeen: 2026-10-01T00:00:00Z
rice: [1, 2, 0.8, 1]
priority: 13
---

# Compile-only verbs can write to disk

`BlueprintHandlerUtils::CompileBlueprintWithDiagnostics` (`Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp`) calls `FKismetEditorUtilities::CompileBlueprint` with `EBlueprintCompileOptions::SkipGarbageCollection` only. Without `SkipSave`, the engine reads `UBlueprintEditorSettings::SaveOnCompile` and, when it is `SoC_Always` or `SoC_SuccessOnly`, saves the package through `FEditorFileUtils::PromptForCheckoutAndSave` (UE 5.8 `BlueprintCompilationManager.cpp:459-490`).

`blueprint.compile`'s wiki page says the verb "never writes to disk" and that persistence goes through `asset.save`, where the Blueprint integrity gate runs. With the preference on, every verb routed through the helper (`blueprint.compile`, `add_variable`, `compile_bpir`, the SCS mutators, ...) saves the package and bypasses that gate. Default preference is `SoC_Never`, so this only bites users who changed it.

**Fix:** pass `SkipSave` in the shared helper for every caller. `blueprint.compile_batch` already opts in via `FCompileDiagnosticsOptions::bSkipSaveOnCompile` (F-blueprint-compile-batch); making it unconditional would remove that option.

## History
- `#1-found-during-compile-batch` `OPEN` developer — Found while implementing F-blueprint-compile-batch: the batch opts into `SkipSave`, but the single-path helper default still honors the preference. Not reproduced live; read from the engine source.
