---
id: B-bind-dispatcher-event-stripped
title: "`compile_bpir` silently drops `event:` and `target:` args from `bind_dispatcher`"
status: DONE
severity: Critical
category: bug
tags: []
---

# `compile_bpir` silently drops `event:` and `target:` args from `bind_dispatcher`

`compile_bpir` silently strips both the `event:` handler ref and (in self-target case) the `target:` object from `bind_dispatcher` statements. The emitted node has neither wired, so BP-compile fails with: `Event Dispatcher pin is not connected  Bind Event to <DispatcherName>`.

**Repro (self-target):**
```
entry custom_event HandleStateChanged(EReplaySaveState NewState) {
    call PrintString(InString: "handler state")
}

entry override Construct() {
    bind_dispatcher OnSaveStateChanged(target: self, event: @HandleStateChanged)
}
```
BPIR compile: 5 nodes, UpToDateWithWarnings. Decompile shows `bind_dispatcher OnSaveStateChanged()` — **both** `target:` and `event:` stripped. BP-compile fails.

**Repro (external target):**
```
entry override Construct() {
    %r = call GetAllWidgetsOfClass(WorldContextObject: self, WidgetClass: /App/.../Handler_C, TopLevelOnly: false)
    %loop = foreach(%r.FoundWidgets) [body -> @body, completed -> @done]
@body:
    %c = cast<Handler_C>(%loop.ArrayElement) [success -> @ok]
@ok:
    bind_dispatcher OnSaveStateChanged(target: %c.AsHandler, event: @HandleStateChanged)
    exec -> @done
@done:
}
```
Decompile shows `bind_dispatcher OnSaveStateChanged(Target: %n2.AsHandler)` — target preserved, `event:` stripped.

This matches the syntax in Example 16 of `bpir-examples.md`, which is presented as working.

**Distinct from `B-bind-dispatcher-self`** (IN-REVIEW): that issue was about `CreateDelegate.self` wiring to external target instead of self. This issue is about `event:` never making it into `CreateDelegate.SelectedFunction` at all — an earlier step silently drops it before the self-wiring fix runs.

**Workaround:** Manual drag-and-drop binding in the BP editor.

## History
- `#1-initial-repro` `OPEN` reporter — Repro'd on scratch widget `W_McpReplayProbe` and on `W_ReplaySaveHandler` during replay subtask 02 probing. Self-target and external-target both affected. `event:` arg stripped unconditionally.
- `#2-prescan-pin-name-fix` `IN-REVIEW` developer — Fixed BpirCompiler.cpp pre-scan (~lines 3534–3541) to accept both UE pin names (`Delegate`, `Target`) and BPIR keywords (`event`, `target`) case-insensitively, populating `DelegateArgValue` / `TargetArgValue` so `CDNode->SetFunction` and target wiring run. Added decompiler-side `event: @FnName` emission in BpirTextEmitter.cpp `EmitDispatcherNode` (~line 775) by reading the linked `UK2Node_CreateDelegate::GetFunctionName()` — restores round-trip fidelity.
- `#3-sigil-strip-followup` `IN-REVIEW` developer — Follow-up fix: initial verification showed `event: @Handler` (the canonical BPIR form per `bpir-examples.md`) stored the function name as literal `"@Handler"` on `UK2Node_CreateDelegate`, producing `@@Handler` in decompile. Added `@` sigil strip in `BpirCompiler.cpp` BindDispatcher emit before `CDNode->SetFunction(FName(*FunctionName))`. Now compiles cleanly and decompile round-trips as single `@Handler`.
- `#4-verified-round-trip` `DONE` tester — Verified via live MCP on `W_ReplaySaveHandler`: `bind_dispatcher OnSaveStateChanged(target: self, event: @McpProbeHandler)` compiles clean (`compiled: true, errors: []`); decompile emits `event: @McpProbeHandler` (single `@`, round-trip intact). Regression tests added: `FCompilerIntegrationBindDispatcherEventKeywordTest`, `FCompilerIntegrationBindDispatcherEventAtSigilTest`, `FCompilerIntegrationDecompilerEmitsEventAtTest` in `TestCompilerDispatchersOps.cpp`.
