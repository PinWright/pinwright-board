---
id: B-stale-function-cache
title: "`insert_bpir_at_node` crash in FCodeFunctionResolver during broad function search"
status: DONE
severity: Critical
category: bug
tags: []
---

# `insert_bpir_at_node` crash in FCodeFunctionResolver during broad function search

`insert_bpir_at_node` crashes with an access violation in `FCodeFunctionResolver::ResolveFunction` → `UClass::FindFunctionByName` → `TSet::FindIndexByHash`. Root cause: `FPluginState::ScannedFunctionLibraries` caches raw `UClass*` pointers from a one-time `TObjectIterator` scan with no GC references. If any cached class is garbage collected or replaced (by BP recompilation, hot-reload, or asset unloading), the pointers become dangling. The existing null check doesn't catch dangling (non-null but invalid) pointers.

**Stack trace (key frames):**
```
TSet::FindIndexByHash(uint, FName&)
UClass::FindFunctionByName(FName, Type)
FCodeFunctionResolver::ResolveFunction(UClass*, FString&)
FCodeFunctionResolver::ResolveFunctionAcrossLibraries(FString&)
FBpirCompiler::EmitInstruction(...)
FBpirCompiler::InsertCodeAfterNode(...)
```

**Fix:** Two-layer defense: (1) `FCodeFunctionResolver::ResolveFunction` guards with `IsValid()` + `CLASS_NewerVersionExists` check before `FindFunctionByName`. Constructor and `BroadFunctionCache` also validate. (2) `FPluginState` invalidates `ScannedFunctionLibraries` on `FCoreUObjectDelegates::ReloadCompleteDelegate` (fires for both Live Coding and classic HotReload), and scan loop filters `CLASS_NewerVersionExists`.

## History
- `#1-crash-minimap-marker-wiring` `OPEN` reporter — Crash during photo inspection minimap marker wiring. BPIR body: `%tint = call MakeColor(R: 1.0, ...)` after `cast<PhotoInspectionTrack>`. First attempt timed out (14795ms), second attempt crashed editor.
- `#2-two-layer-stale-cache-fix` `IN-REVIEW` developer — Two-layer fix: (1) `FCodeFunctionResolver::ResolveFunction` guards with `IsValid()` + `CLASS_NewerVersionExists` before calling `FindFunctionByName` — protects all callers. Constructor filters stale classes from FPluginState scan. `ResolveFunctionBroadSearch` validates and evicts stale `UFunction*` entries. (2) `FPluginState` invalidates `ScannedFunctionLibraries` on `FCoreUObjectDelegates::ReloadCompleteDelegate`. Scan loop filters `CLASS_NewerVersionExists`. 4 safety tests added.
- `#3-smoke-test-insert-bpir` `DONE` tester — Smoke test on `W_McpVerifyTemp`: `insert_bpir_at_node` on HandleClick function entry with body `%color = call MakeColor(R:1.0,G:0.5,B:0.25,A:1.0); call PrintString(InString:"Inserted");`. Exercised the `FCodeFunctionResolver` + `BroadFunctionCache` path (MakeColor lives in a kismet library found via scanned-function-libraries). Completed without crash, `compiled: true`, 2 nodes created. Editor stable across subsequent calls. Note: the GC'd-UClass scenario that originally triggered the crash is non-deterministic to reproduce via MCP; the 4 automation tests exercise it directly. Happy-path non-regression confirmed.
