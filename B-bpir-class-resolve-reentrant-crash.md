---
id: B-bpir-class-resolve-reentrant-crash
title: "`compile_bpir` crashes editor via re-entrant BP compile from `ResolveUClass` → `StaticLoadObject`"
status: WONTFIX
severity: Critical
category: bug
tags: [bpir, crash, class-resolution, reentrancy, async-load]
---

# `compile_bpir` crashes editor via re-entrant BP compile from `ResolveUClass` → `StaticLoadObject`

Calling `blueprint.compile_bpir` on a BPIR body that references an unloaded widget class crashes the editor with a `FindPinChecked` fatal assertion inside `FKismetCompilerContext::ReplaceConvertibleDelegates`.

The stack trace (see below) shows the chain: `FBpirCompiler::WireDataPins` → `ResolveUClass` → `StaticLoadObject` synchronously loads the referenced package → `FlushAsyncLoading` fires → dependent `FBlueprintCompilationManagerImpl::FlushCompilationQueueImpl` runs mid-wire → `ReplaceConvertibleDelegates::FindPinChecked` crashes because the BPIR-in-progress graph has delegate nodes in an intermediate state (pins not yet wired).

**Repro (confirmed on live editor, UE 5.6):**

In `W_LyraFrontEnd` (or any already-loaded widget BP) compile:

```
entry widget_event W_ReplayEditorButton.OnClicked() {
    %n0 = call GetOwningPlayer()
    %n1 = call K2Node_AsyncAction(OwningPlayer: %n0, WidgetClass: /App/App/UI/LobbyAndMenu/W_MyReplaySelect.W_MyReplaySelect_C, LayerName: (TagName="UI.Layer.Menu"), bSuspendInputUntilComplete: true) [AfterPush -> @afterpush]

@afterpush:
    %n2 = cast<W_MyReplaySelect_C>(%n1.UserWidget) [success -> @ok]

@ok:
    clear_dispatcher OnOpenReplayRequested(Target: %n2.AsWMyReplaySelect)
    bind_dispatcher OnOpenReplayRequested(Target: %n2.AsWMyReplaySelect, Delegate: $OutputDelegate)
}

entry custom_event OnOpenReplayRequested_Event(struct<EditorReplay> Replay) {
    call OpenReplay(Target: $W_ReplayOpenHandler, Replay: $Replay)
}
```

Preconditions that contribute:
- `W_MyReplaySelect_C` (referenced by `cast<>` and `K2Node_AsyncAction` `WidgetClass`) is not already loaded in the editor session.
- A `bind_dispatcher` on a `DYNAMIC_MULTICAST_DELEGATE_OneParam` with a USTRUCT (`const FEditorReplay&`) parameter is present in the same compile.
- A `$OutputDelegate` delegate-pin reference is used (the standard BPIR pattern for dispatcher-to-event wiring).

Observed behavior: first invocation returned `UE RPC request timed out after 14836ms`; second invocation crashed the editor.

**Stack (abbreviated):**

```
UEdGraphNode::FindPinChecked(FName, EEdGraphPinDirection) EdGraphNode.h:586
FKismetCompilerContext::ReplaceConvertibleDelegates(UEdGraph*) KismetCompiler.cpp:5971
FKismetCompilerContext::CreateAndProcessUbergraph() KismetCompiler.cpp:4125
FKismetCompilerContext::CreateFunctionList() KismetCompiler.cpp:4686
FWidgetBlueprintCompilerContext::CreateFunctionList() WidgetBlueprintCompiler.cpp:236
FKismetCompilerContext::CompileClassLayout(EInternalCompilerFlags) KismetCompiler.cpp:4889
FBlueprintCompilationManagerImpl::FlushCompilationQueueImpl(...) BlueprintCompilationManager.cpp:1447
FBlueprintCompilationManager::FlushCompilationQueue(...) BlueprintCompilationManager.cpp:4227
FScopedClassDependencyGather::~FScopedClassDependencyGather() BlueprintSupport.cpp:502
FLinkerLoad::CreateExport(int) LinkerLoad.cpp:5742
...
LoadPackageInternal(...) UObjectGlobals.cpp:1771
...
StaticLoadObject(...) UObjectGlobals.cpp:1466
ResolveUClass(const FString&) ClassUtils.cpp:112
FBpirCompiler::WireDataPins(int, FBpirInstruction&, FBpirEntryBlock&) BpirCompiler.cpp:4478
FBpirCompiler::Compile(const FString&, EBpirCompileMode) BpirCompiler.cpp:1096
```

**Impact:** any BPIR that references a widget/blueprint class by path and that class is not currently loaded (very common — Content Browser doesn't keep every widget loaded) risks either timeout or editor crash. For content-rich projects this makes `widget_event` + `bind_dispatcher` workflows unreliable for any cross-widget interaction that isn't immediate.

**Workaround:** preload the referenced class(es) before calling `compile_bpir` (e.g., open the widget asset, or issue a `blueprint.inspect` / `asset.get` call on each referenced class to force it into memory). Crude, easy to forget.

**Proposed fix:** `FBpirCompiler` should resolve all external `UClass` references in a pre-pass *before* any pin wiring begins, using a load path that cannot trigger `FlushCompilationQueue` re-entrantly — e.g., defer class loads outside any KismetCompiler scope, or use `LoadObject` with flags that avoid the BP-compile cascade. Alternatively, wrap the `ResolveUClass` calls inside `WireDataPins` with a `FBlueprintCompileReinstancer::FBlueprintCompileReinstancerDeferralScope` (or equivalent) so mid-wire loads can't trigger the dependent-compile flush.

## History
- `#1-initial-repro` `OPEN` reporter — Crashed live editor during replay subtask #4 BP wire-up while building the `W_ReplayEditorButton.OnClicked` → `W_MyReplaySelect.OnOpenReplayRequested` → `W_ReplayOpenHandler.OpenReplay` flow on `W_LyraFrontEnd`. First call: 14.8 s RPC timeout. Second call: editor crashed with `FindPinChecked` fatal in `ReplaceConvertibleDelegates` after async-load of `W_MyReplaySelect_C` triggered re-entrant compilation flush mid-wire. Full stack captured above. Reported by user.
- `#2-inspect-also-crashes` `OPEN` reporter — After user-initiated editor restart, reproduced via a different MCP entry point: `blueprint.inspect` on `/App/App/UI/LobbyAndMenu/W_MyReplaySelect` also crashes with the identical `ReplaceConvertibleDelegates::FindPinChecked` assertion. This confirms the crash is not `compile_bpir`-specific — **any MCP tool that triggers `StaticLoadObject` on `W_MyReplaySelect_C` crashes the editor**. Specifically, `BlueprintInspectHandler.cpp:39` calls `LoadBlueprintAsset` → `StaticLoadObject` → `FlushAsyncLoading` → `FBlueprintCompilationManagerImpl::FlushCompilationQueueImpl` → crash. Root cause is more general than the original report: any tool-initiated sync-load of a BP that carries a "convertible delegate" (dynamic multicast delegate bound in its own BP graph to a mismatched signature — common after C++ delegate schema changes) can detonate the compilation manager. Workaround that works: bypass MCP BP edits entirely for the affected BP and do the wiring in C++.
- `#3-preload-external-classes` `IN-REVIEW` developer — Added `FBpirCompiler::PreloadExternalClasses` pre-pass in `BpirCompiler.{h,cpp}`. Walks `Inst.TypeArg` and every `Arg.Value` across all parsed `FBpirEntryBlock`s after `Parser.Parse`/`Parser.ParseBody` succeeds and before any graph mutation, force-loading candidates via `ResolveUClass` (`Utils/ClassUtils.cpp:101`). Candidates are filtered to skip empty/`self`/`%ref`/`$var`/`::`-enum/quoted/struct-literal/pure-numeric strings and deduplicated via `TSet`. Called at all three compile entry points — `Compile(Code, Mode)`, `InsertCodeAfterNode`, `CompileBodyIntoGraph`. Result: mid-wire `ResolveUClass` calls now hit `FindObject` fast paths, so `LoadObject` → `FlushAsyncLoading` → `FlushCompilationQueue` reentrance during Emit/Wire is eliminated.
- `#4-decompile-also-crashes` `OPEN` tester — Returned (CRASH): editor crashed from `blueprint.decompile assetPath="/App/App/UI/LobbyAndMenu/W_LyraFrontEnd" graphName="EventGraph"` during MCP verification of other items. First call: `UE RPC request timed out after 14817ms`; during that call the editor crashed. Captured stack is identical to the original report but originating in `BlueprintDecompilerHandler.cpp:58 → LoadBlueprintAsset → StaticLoadObject → FlushAsyncLoading → FBlueprintCompilationManagerImpl::FlushCompilationQueueImpl → FKismetCompilerContext::ReplaceConvertibleDelegates → UEdGraphNode::FindPinChecked` (same `W_LyraFrontEnd` / `CREATEDELEGATE_PROXYFUNCTION_0` / `SKEL_W_LyraFrontEnd_C` signatures as the earlier `compile_bpir` repro). The landed fix is BPIR-compile-only: `FBpirCompiler::PreloadExternalClasses` runs before Emit/Wire on the three `FBpirCompiler::Compile*` entry points, so `compile_bpir` is now defensively safe. But the prior reporter's second OPEN entry already documented: "any MCP tool that triggers `StaticLoadObject` on the affected BP crashes the editor — `BlueprintInspectHandler.cpp:39` calls `LoadBlueprintAsset` → `StaticLoadObject` → `FlushAsyncLoading` → `FBlueprintCompilationManagerImpl::FlushCompilationQueueImpl` → crash." `BlueprintDecompilerHandler.cpp:58` exhibits the identical load path and has no preload / no re-entrancy guard. The fix needs to apply to every handler that calls `LoadBlueprintAsset` / `StaticLoadObject` on a BP path — at minimum `blueprint.decompile`, `blueprint.inspect`, `blueprint.compile`, `blueprint.list` (when loading candidates), and any XML/describe handler that resolves a BP path — not just the BPIR compiler entry points. Either wrap `LoadBlueprintAsset` globally with a reentrancy guard / pre-load that flushes the compilation queue OUTSIDE the KismetCompiler scope, or move the preload into `LoadBlueprintAsset` itself in `Utils/AssetUtils.cpp:592` so every caller is covered.
- `#5-wontfix` `WONTFIX` user — Closed. Creation-side protection already landed (`#3-preload-external-classes` IN-REVIEW for BPIR compile paths, plus P0-10 `ScrubStaleUFunctionsFromClass` + `RefreshBpirDelegateNodes` from `B-bp-saved-state-corruption-mcp-edits`): BPIR authoring starting from a clean BP no longer produces the `CREATEDELEGATE_PROXYFUNCTION_*` + stale-`MemberReference` state that makes `ReplaceConvertibleDelegates` crash `FindPinChecked` on cold reload. The remaining surface the tester's `#4-decompile-also-crashes` return flagged — `blueprint.decompile` / `blueprint.inspect` / any `LoadBlueprintAsset` caller touching an ALREADY-CORRUPT `.uasset` — is a load-site problem on BPs that were corrupted in a prior session. Those .uassets are already broken on disk; no amount of plugin-side re-entrancy guarding or preload-flushing can undo serialized bad state once it's there. Recovery path for corrupt assets is git-restore or rebuild-from-scratch, not further plugin fixes. Future-new corruption is prevented by the BPIR-side fixes; historical corruption stays a user-action problem. Closing as WONTFIX.
