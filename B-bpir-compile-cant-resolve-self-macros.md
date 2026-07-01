---
id: B-bpir-compile-cant-resolve-self-macros
title: "BPIR compile_bpir can't resolve user-defined (self) macros — only searches StandardMacros library"
status: DONE
severity: High
category: bug
tags: [bpir, compiler, round-trip, macro, self-macro]
---

# `compile_bpir` can't reference user-defined macros on the same Blueprint

`blueprint.decompile` emits `macro <Name>(args)` for user-defined macro
instances (e.g. `%n2 = macro LoadMission(UserFacingExp: $DA_Track, Mode: $Mode)`),
matching the documented BPIR syntax for macro invocations. Re-feeding that
text to `blueprint.compile_bpir` fails with:

```
COMPILE_FAILED: Macro 'LoadMission' not found in library 'StandardMacros'
```

The resolver in `CodeNodeEmitter::CreateMacroNode`
(`Source/EditorAutomationRpcGateway/Private/Compiler/CodeNodeEmitter.cpp:394-416`)
hardcodes `GetStandardMacrosLibrary()` as the only search target:

```cpp
UK2Node_MacroInstance* FCodeNodeEmitter::CreateMacroNode(FName MacroName, UEdGraphPin*& InOutExecPin)
{
    ...
    UBlueprint* MacroLib = GetStandardMacrosLibrary();
    UEdGraph* MacroGraph = MacroLib ? FindMacroGraph(MacroLib, MacroName) : nullptr;
    ...
}
```

User-defined macros that live on the target Blueprint's own `MacroGraphs`
list (or on a referenced project-side macro library Blueprint) are never
checked. Decompile → edit → compile round-trip is broken for every
Blueprint that defines its own macros (very common in widget-side
"`LoadMission` / `Update` / `Refresh`" patterns across this project).

Same shape as the now-DONE `B-bpir-subsystem-getter-roundtrip`: decompile
emits an identifier the compiler can't ingest. Different surface — that
one was a missing decompile keyword; this is a missing compile resolver.

## Repro

1. Decompile `W_MissionSelectButton::Button.OnClicked`
   (`blueprint.decompile`). The body contains
   `%n2 = macro LoadMission(UserFacingExp: $DA_Track, Mode: $Mode)`.
2. Feed that body back through `blueprint.compile_bpir`.
3. `COMPILE_FAILED: Macro 'LoadMission' not found in library 'StandardMacros'`.

The compiler comment at `BpirCompiler.cpp:4112` even mentions "Pre-existing
user function / macro graphs on the target BP that the BPIR code calls"
as a supported case for the resolution loop, but the macro emitter doesn't
honor it.

## Impact

- Breaks decompile → edit → compile iteration for any Blueprint that
  defines its own macros — pervasive across PDS widget code
  (`W_MissionSelectButton`, `W_MyLessonsListItem`,
  `W_TutorialSelectButton`, `W_TornamentSelectButton`, many more all
  define a per-widget `LoadMission` macro plus utility macros).
- Combined with `B-bpir-macro-recompile-orphans-caller-instances`,
  any session that touches a self-macro creates a workflow trap:
  recompile the macro → caller breaks silently → can't restore the
  caller via BPIR because the macro reference doesn't resolve.
- Forces callers to inline the macro body, which loses round-trip
  fidelity and bloats the caller graph.

## Workaround

Inline the macro body directly into each caller. The standalone macro
becomes unused but harmless; the launch logic moves into the click
handlers / event entries.

## Fix

Resolve bare `macro Name(...)` references in two tiers: search
`StandardMacros` first, then search the target Blueprint's own
`MacroGraphs`, excluding the graph currently being compiled. Keep project
macro-library discovery out of scope for this syntax so local resolution
stays deterministic and existing StandardMacros names keep their current
priority.

Update the macro-not-found diagnostic to name both searched locations.
Add a regression test that creates a Blueprint with local macro `MyMacro`,
compiles an event that calls `macro MyMacro()`, and asserts the emitted
`UK2Node_MacroInstance` is bound to that same local macro graph.

## History
- `#1-initial-repro` `OPEN` reporter — Session recompiling `W_MissionSelectButton::LoadMission` macro via `compile_bpir` orphaned its caller `Button.OnClicked` (see `B-bpir-macro-recompile-orphans-caller-instances`). Attempting to restore the caller by re-running `compile_bpir` on Button.OnClicked with the decompiled `macro LoadMission(UserFacingExp: $DA_Track, Mode: $Mode)` invocation produced `COMPILE_FAILED: Macro 'LoadMission' not found in library 'StandardMacros'`. `CodeNodeEmitter.cpp:401-402` confirms only `GetStandardMacrosLibrary()` is checked. Same self-macro shape repeats on `W_MyLessonsListItem::BP_OnClicked` (`macro LoadMission(UserFacingExp: %n6)`). Workaround: inline the macro body into each caller, abandoning round-trip fidelity.
- `#2-resolve-self-macros` `IN-REVIEW` developer — `CodeNodeEmitter.cpp` now resolves bare macro calls through StandardMacros first and same-Blueprint `MacroGraphs` second, excluding the graph currently being compiled; `BpirCompiler.cpp` reports both searched locations on failure. Added `FBpirCompilerSelfMacroResolutionTest` in `TestCompilerSelfMacros.cpp` to compile a local `MyMacro` and assert the caller `UK2Node_MacroInstance` binds to that graph.
- `#3-skip-no-mcp-access` `SKIP` tester — editor-automation MCP tool not exposed in verifier subagent session, can't exercise live `blueprint.compile_bpir`. Static code review confirms the claim: `CodeNodeEmitter.cpp:409-414` calls `GetStandardMacrosLibrary()` first then `BpirCompilerMacroUtils::FindMacroGraphByName(Blueprint, MacroName, Graph)` (passing current Graph as ExcludedGraph); helper signature at `CodeNodeEmitter.h:44-45` accepts `ExcludedGraph`; `BpirCompiler.cpp:5210` diagnostic reads `"Macro '%s' not found in StandardMacros or target Blueprint MacroGraphs"`; regression test `FBpirCompilerSelfMacroResolutionTest` at `TestCompilerSelfMacros.cpp:40-110` compiles local `MyMacro`, then compiles caller `macro MyMacro()` from BeginPlay, then asserts the resulting `UK2Node_MacroInstance->GetMacroGraph()` equals the local graph.
- `#4-verify-live-self-macro` `DONE` tester — Verified live via direct HTTP RPC against `/App/App/UI/LobbyAndMenu/Elements/W_MissionSelectButton` (which still owns a `LoadMission` macro graph, confirmed by `blueprint.decompile graphName=LoadMission`). Posted `blueprint.compile_bpir` (mode `append`) with `entry custom_event McpVerifySelfMacro(... ) { %n0 = macro LoadMission(UserFacingExp: $Exp, Mode: $M) }` — response `compiled: true`, `nodeCount: 2`, `errors: []`, no `Macro 'LoadMission' not found ...` diagnostic. The created K2Node_MacroInstance reported `title: "Load Mission"` on delete, confirming the resolver bound to the same-Blueprint `MacroGraphs` entry. Cleaned up by deleting the created entry node (cascade-removed the orphan macro instance).
