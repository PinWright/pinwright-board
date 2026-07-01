---
id: B-bpir-asyncaction-disambiguation
title: "`call K2Node_AsyncAction_*` resolves to wrong subclass"
status: DONE
severity: Medium
category: bug
tags: []
---

# `call K2Node_AsyncAction_*` resolves to wrong subclass

`compile_bpir` matches `K2Node_AsyncAction_*` by the root class name, so any BPIR code referencing a particular async action subclass resolves to an unrelated one (typically `AsyncLoadPrimaryAsset` — first in the type tree).

**Repro:**
```
entry custom_event TestPushLayer() {
    %owner = call GetOwningPlayer()
    %w = call K2Node_AsyncAction(OwningPlayer: %owner, WidgetClass: ..., LayerName: ..., bSuspendInputUntilComplete: false) [AfterPush -> @done, BeforePush -> @done, then -> @done]
@done:
}
```
→ `Could not find target pin 'OwningPlayer' on node 'AsyncLoadPrimaryAsset'` — wrong subclass picked.

Full name form:
```
%w = call K2Node_AsyncAction_PushContentToLayerForPlayer(...) [...]
```
→ `Unresolved function: 'K2Node_AsyncAction_PushContentToLayerForPlayer'`

**Expected:** BPIR should accept fully-qualified K2Node subclass names, OR disambiguate the root `K2Node_AsyncAction` match based on the pin name set provided.

**Workaround:** None found via BPIR. Manual BP editor placement only.

## History
- `#1-initial-repro` `OPEN` reporter — Hit while trying to push `W_Error` popup from `W_ReplaySaveHandler`. Both generic and specific K2Node forms failed.
- `#2-two-pass-name-match` `IN-REVIEW` developer — Restructured `AutoConfigureAsyncTaskNode` in CodeNodeEmitter.cpp (~lines 1089–1115) to a two-pass match: exact case-insensitive `ActionCoreName.Equals(NodeCoreName)` first, StartsWith fallback only if no exact match. Ambiguous cases (>1 candidate in either pass) now emit an error listing candidates rather than silently picking the first. Fixes `K2Node_AsyncAction_PushContentToLayerForPlayer` mis-routing to `AsyncLoadPrimaryAsset`.
- `#3-pin-set-matching` `IN-REVIEW` developer — Initial two-pass name-heuristic fix was insufficient: the synthetic `K2Node_AsyncAction_<Name>` form was never a real UClass, so `FindFirstObjectSafe<UClass>` failed before any matching ran. Replaced the entire name-heuristic matcher with pin-set-based matching: `AutoConfigureAsyncTaskNode(UK2Node_BaseAsyncTask*, const TArray<FString>& ProvidedArgNames)` picks the `UBlueprintAsyncActionBase` subclass whose static factory function's parameter set is a superset of the provided BPIR arg names (tightest fit wins; ties emit an ambiguity error). `CreateGenericK2Node` detects synthetic `K2Node_AsyncAction_<anything>` and redirects to `UK2Node_AsyncAction::StaticClass()`; the pin-set matcher disambiguates. Both prefix-strip logic (`K2Node_AsyncAction_`, `K2Node_`) and the `ExplicitNodeCoreName` hint parameter deleted. Mirrors UE's palette-time model (one pre-configured UK2Node_AsyncAction per subclass).
- `#4-verified-pin-set-match` `DONE` tester — Verified via live MCP on `W_McpVerifyTemp`: both `call K2Node_AsyncAction(OwningPlayer:..., WidgetClass:..., LayerName:..., bSuspendInputUntilComplete:false)` (generic root) and `call K2Node_AsyncAction_PushContentToLayerForPlayer(...)` (synthetic) compile clean; pin inspection confirms correct subclass picked (pins: `BeforePush`, `AfterPush`, `UserWidget`, `OwningPlayer`, `WidgetClass`, `LayerName`, `bSuspendInputUntilComplete` from `AsyncAction_PushContentToLayerForPlayer`). Regression tests added: `FCompilerIntegrationAsyncActionPinSetMatchingTest`, `FCompilerIntegrationAsyncActionSyntheticRedirectTest` in `TestCompilerDispatchersOps.cpp` (asserts `Factory->GetOwnerClass() == UAsyncTaskDownloadImage::StaticClass()` — strict check catches the pre-fix `AsyncLoadPrimaryAsset` regression).
