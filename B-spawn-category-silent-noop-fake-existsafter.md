---
id: B-spawn-category-silent-noop-fake-existsafter
title: "`debug.spawn_category` is a silent no-op that always returns `commandExecuted:false` + hardcoded `existsAfter:true` — a bogus category name is indistinguishable from a real toggle"
status: IN-REVIEW
severity: Medium
category: bug
tags: [debug, gameplay-debugger, spawn-category, silent-noop, success-no-effect, hardcoded-field, existsafter, commandexecuted]
---

# `debug.spawn_category` reports success-with-no-effect and lies about `existsAfter`

`debug.spawn_category` is documented as "Toggle a UE Gameplay Debugger category on/off (equivalent of pressing the category key while the debugger is open). Wraps the 'GameplayDebuggerCategory <name>' console command." In practice the handler (`DebugHandler.cpp`) does `GEngine->Exec(nullptr, "GameplayDebuggerCategory <name>")` and then `SendSuccess` unconditionally, with two concrete defects:

1. **`existsAfter` is a hardcoded literal `true`** (`DebugHandler.cpp:24` — `Resp->SetBoolField(TEXT("existsAfter"), true);`). The field name promises a verification ("the category exists after the toggle"), but it is a constant: it is `true` even for a garbage category name that was never registered. This is a result misreport — the one field a caller would use to confirm the toggle landed is a lie.

2. **`commandExecuted` is `false` for every input, yet the RPC still reports success.** `GEngine->Exec` returns false because the wrapped string `GameplayDebuggerCategory <name>` is **not a registered console command in any scope** — there is no `GameplayDebuggerCategory` console command in the engine. The actual gameplay-debugger console commands are `gdt.ToggleCategory <CategoryIdx>` (numeric index) and `gdt.EnableCategoryName <CategoryNamePattern>` (name pattern), both registered in `GameplayDebuggerLocalController.cpp`. So `commandExecuted:false` is returned for *every* input — valid category, wrong-case name, or garbage — because the command name itself is unknown, not because of an editor-scope no-op. The toggle does nothing, but the RPC `SendSuccess`es regardless. A valid category, an invalid category, and a typo are all indistinguishable: every one returns `{"commandExecuted":false,"existsAfter":true}` with no error.

Net effect: a caller asking "did the Perception/BehaviorTree/EQS debugger overlay turn on?" gets a fully success-shaped response (`existsAfter:true`) for ALL of (a) a real category, (b) a real category name with the wrong case, and (c) a category that does not exist at all — none of which ever toggled anything, because the wrapped command never executes. There is no JSON-RPC signal distinguishing any of these — the success-shaped echo masks both the unknown-command no-op and outright bad input.

## Replay-confirmed repro (live editor, `mcp__editor-automation__call`)

- `debug.spawn_category {categoryName:"Perception"}`
  → `{"categoryName":"Perception","consoleCommand":"GameplayDebuggerCategory Perception","commandExecuted":false,"existsAfter":true}`
- `debug.spawn_category {categoryName:"BehaviorTree"}`
  → `{"categoryName":"BehaviorTree","consoleCommand":"GameplayDebuggerCategory BehaviorTree","commandExecuted":false,"existsAfter":true}`
- `debug.spawn_category {categoryName:"EQS"}`
  → `{"categoryName":"EQS","consoleCommand":"GameplayDebuggerCategory EQS","commandExecuted":false,"existsAfter":true}`
- **Control — a category that does not exist:** `debug.spawn_category {categoryName:"ThisCategoryDoesNotExist_XYZ123"}`
  → `{"categoryName":"ThisCategoryDoesNotExist_XYZ123","consoleCommand":"GameplayDebuggerCategory ThisCategoryDoesNotExist_XYZ123","commandExecuted":false,"existsAfter":true}`

The control proves `existsAfter:true` is a hardcoded constant (identical response shape for a guaranteed-bogus name) and that `commandExecuted:false` carries no validity information — both real and bogus categories produce the same success-shaped result.

## Root cause (`DebugHandler.cpp`)

Lines 17–26:
```cpp
FString Cmd = FString::Printf(TEXT("GameplayDebuggerCategory %s"), *CategoryName);
bool bSuccess = GEngine->Exec(nullptr, *Cmd);
...
Resp->SetBoolField(TEXT("commandExecuted"), bSuccess);
Resp->SetBoolField(TEXT("existsAfter"), true);   // hardcoded — never reflects reality
Ctx.SendSuccess(Resp);
```
`existsAfter` is never computed; `bSuccess` (always false here, because `GameplayDebuggerCategory` is not a real console command) is reported but never gates the success/error decision.

## Impact

The discovery-obvious "turn on AI debug overlay, confirm it's on" round-trip cannot be confirmed: the response says `existsAfter:true` regardless, and the wrapped command never actually runs (it is not a registered command). A typo'd or non-registered category name silently succeeds, so callers chasing "why isn't my overlay showing" get no signal that anything was wrong (the wiki example even says 'Behavior' while the engine-registered name is 'BehaviorTree' — but even the correct name returns the same clean success because the command string itself is bogus). The hardcoded `existsAfter` is part of the recurring `existsAfter:true` silent-noop family on this board (e.g. `B-configure-sense-config-silent-noop`, `B-create-level-saved-true-no-umap`) but is a distinct handler/method not covered by any existing ticket.

**Workaround:** none reliable from the RPC alone — the response cannot confirm the toggle, and the wrapped command does not exist. The closest real engine commands are `gdt.ToggleCategory <CategoryIdx>` / `gdt.EnableCategoryName <name>`, but both require a live game world (a `UGameplayDebuggerLocalController` with a valid `CachedReplicator`), which the editor-scope RPC has no access to. Toggling the gameplay debugger reliably needs a running PIE session driven outside this RPC.

**Fix:** Stop fabricating success. The handler must NOT `SendSuccess` with a hardcoded `existsAfter:true` for a command that never executed. Replace the silent-success stub with a loud failure that the caller can act on:
- Remove the hardcoded `existsAfter:true` literal entirely (it cannot be trusted and never reflected reality).
- Since `GEngine->Exec(nullptr, "GameplayDebuggerCategory <name>")` always returns false (unknown command, no world bound), `SendError` with a clear code (e.g. `NOT_IMPLEMENTED` / `COMMAND_FAILED`) explaining that the gameplay-debugger toggle could not be executed from editor scope and naming the real commands (`gdt.ToggleCategory` / `gdt.EnableCategoryName`) and their live-world requirement. Never return a success-shaped response that masks the no-op.

A fuller future enhancement (separately, if a name-validation surface becomes available) could look the category up in the Gameplay Debugger addon registry and only attempt the (correctly named, world-bound) command when a PIE world exists — but the immediate defect is the fabricated-success result, which must fail loud. The wrong-example / discovery angle (the 'Behavior' literal, a `debug.list_categories` surface) is owned by the sibling ticket `E-spawn-category-name-discovery`.

## History
- `#1-initial-repro` `OPEN` reporter — Realism-mode fuzz of the gameplay-debugger overlay workflow (`debug.spawn_category` for Perception/BehaviorTree/EQS). Replay-confirmed live: all three valid categories AND a guaranteed-bogus name `ThisCategoryDoesNotExist_XYZ123` return identical `{"commandExecuted":false,"existsAfter":true}` success — proving `existsAfter:true` is a hardcoded literal (`DebugHandler.cpp:24`) and `commandExecuted:false` carries no validity signal. The RPC `SendSuccess`es regardless of whether the category exists or the toggle landed (editor scope, no PIE world consumes the exec), so bad input and a real no-op are indistinguishable. Not a duplicate: no existing board ticket mentions `spawn_category` / `GameplayDebuggerCategory` / `commandExecuted`; sibling silent-noop tickets (`B-configure-sense-config-silent-noop`, `B-create-level-saved-true-no-umap`) are different methods/namespaces.
- `#2-reword-mechanism` `OPEN` developer — Reworded the body/Fix to correct the failure mechanism. The original `#1` framing blamed `commandExecuted:false` on "editor scope / no PIE / no PlayerController consuming the exec" and proposed a PIE workaround / a Fix Option 3 about the toggle "only taking effect with a live game world." Source check: there is NO `GameplayDebuggerCategory` console command anywhere in the engine (full grep of `C:\UE_5.7\Engine\Source` for that literal matches only C++ class names, never a `FAutoConsoleCommand` registration). The only registered gameplay-debugger toggles are `gdt.ToggleCategory <CategoryIdx>` (numeric index) and `gdt.EnableCategoryName <CategoryNamePattern>`, both in `Runtime/GameplayDebugger/Private/GameplayDebuggerLocalController.cpp` (`gdt.EnableCategoryName` at ~1260, `gdt.ToggleCategory` at ~1254). So `GEngine->Exec` returns false because the wrapped string is an UNKNOWN command — it would return false in PIE too — not because of an editor-scope no-op. Rewrote defect #2, the Impact paragraph, the Workaround, and collapsed the three fix options into a single fail-loud Fix (the wrong-example/discovery angle stays owned by `E-spawn-category-name-discovery`). Severity (Medium) and category (bug) unchanged.
- `#3-fail-loud-fix` `IN-REVIEW` developer — Implemented the fail-loud fix in `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Handlers/Debug/DebugHandler.cpp`. Deleted the hardcoded `existsAfter:true` literal entirely and gated the response on the exec result: since `GEngine->Exec(nullptr, "GameplayDebuggerCategory <name>")` always returns false (unknown command, no world bound), the handler now `SendError("NOT_IMPLEMENTED", ...)` with a message naming the real commands (`gdt.ToggleCategory` / `gdt.EnableCategoryName`) and their live-world requirement, instead of `SendSuccess` with a fabricated constant. Regression coverage added to the existing `Private/Tests/EditorOps/TestDebugHandlers.cpp` (next to the registration-only spawn_category tests): `FDebugSpawnCategoryFailsLoudTest` (`debug.spawn_category.NoFakeExistsAfterFailsLoud`) uses `InvokeHandlerWithCapture` to assert `bSuccess==false`, `ErrorCode=="NOT_IMPLEMENTED"`, and that no `existsAfter:true` appears on the response; `FDebugSpawnCategoryBogusNameFailsLoudTest` (`debug.spawn_category.BogusNameFailsLoud`) pins that a guaranteed-bogus name produces the same loud error (the original control case). Both would fail if the silent-success stub were restored. Not compiled/run here — deferred to the verification phase.
- `#4-method-removed-in-cull` `IN-REVIEW` reporter — Note (superseded by removal): `debug.spawn_category` was removed entirely in the RPC cull recorded in [`E-rpc-cull-151-record`](E-rpc-cull-151-record.md) (it is one of the 55 stubs). The `#3` fail-loud fix is therefore moot — the method no longer exists — and its regression guards (`debug.spawn_category.NoFakeExistsAfterFailsLoud` / `.BogusNameFailsLoud`, plus the existence checks, in `Tests/EditorOps/TestDebugHandlers.cpp`) were deleted with the handler. Removing it emptied `DebugHandler.cpp` and orphaned the `docs/wiki-src/debug.md` namespace overlay. Recorded for traceability.
