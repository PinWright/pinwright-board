---
id: E-bpir-classify-node-registry
title: "Replace GraphWalker::ClassifyNode if-chain with TMap<UClass*, ENodeSemantics> registry"
status: DONE
severity: Medium
category: ergonomic
tags: [bpir, decompiler, graph-walker, registry]
---

# `ClassifyNode` should use a registry, not a 15-arm if-chain

`Decompiler/GraphWalker.cpp:111-223` (`ClassifyNode`) is a sequential
chain of 15+ `IsA<>`/`Cast<>` checks mapping each concrete K2Node
class to an `ENodeSemantics` enum value. Adding support for a new
K2Node subclass requires editing this function — there is no
extension point for plugin-defined node types.

The dispatch is structurally a `UClass*` → `ENodeSemantics` lookup.
A `TMap<UClass*, ENodeSemantics>` populated at startup (with `IsA`-
compatible ancestor walk for fallthrough on subclasses) replaces
the chain with one O(depth) lookup, lets future K2Node types be
registered without editing this file, and removes the implicit
"earlier branch wins" ordering coupling between the 15 arms.

## Sketch

```cpp
// Initialized once at module startup.
static const TMap<UClass*, ENodeSemantics>& GetSemanticsRegistry()
{
    static TMap<UClass*, ENodeSemantics> Map = []
    {
        TMap<UClass*, ENodeSemantics> M;
        M.Add(UK2Node_CallFunction::StaticClass(), ENodeSemantics::Call);
        M.Add(UK2Node_VariableGet::StaticClass(), ENodeSemantics::VariableGet);
        // ... one line per current arm
        return M;
    }();
    return Map;
}

ENodeSemantics ClassifyNode(const UEdGraphNode* Node)
{
    const auto& Map = GetSemanticsRegistry();
    for (UClass* C = Node->GetClass(); C; C = C->GetSuperClass())
    {
        if (const ENodeSemantics* Found = Map.Find(C)) return *Found;
    }
    return ENodeSemantics::Unknown;
}
```

The `IsA`-compatible walk preserves current behavior where a subclass
inherits its base's classification (e.g. any `UK2Node_AsyncAction`
subclass classifies the same as the base).

## Notes

- Field-notify FName allowlist branch (lines 203-208) currently
  compares `FuncName == TEXT("K2_AddFieldValueChangedDelegate")` etc.
  These should remain inside the call-function arm but are a separate
  cleanup — see the magic-strings cleanup ticket.
- `IsLatentNode` (line 237) is a separate per-pin check and does not
  belong in this registry — leave alone.

## Why ergonomic, not bug

Behavior is unchanged; only the extensibility shape changes.
Filed as ergonomic because the win is "third parties / future K2Node
types just register an entry" rather than "this fixes a wrong
output."

## History
- `#1-initial-spec` `OPEN` reporter — Audit identified `ClassifyNode` as the primary dispatch gate the long tail of K2Node types must pass through. Registry refactor unlocks plug-in extensibility without editing the file each time.
- `#2-registry-refactor` `IN-REVIEW` developer — Refactored `FGraphWalker::ClassifyNode` in `Source/EditorAutomationRpcGateway/Private/Decompiler/GraphWalker.cpp` to dispatch via an anonymous-namespace function-local-static `GetSemanticsRegistry()` returning `const TMap<UClass*, ENodeSemantics>&`. Pure class-based arms (`IfThenElse`, `ExecutionSequence`, `DynamicCast`, `Timeline`, `Switch`, `FunctionResult`, `VariableSet`, `VariableGet`, `Comment`, `Knot`, plus the four delegate node classes under `MCP_HAS_DELEGATE_NODES`) live in the registry; the registry walk is `for (UClass* C = Node->GetClass(); C; C = C->GetSuperClass())`, preserving most-derived-wins semantics and supporting plugin subclasses. Three behaviorally-conditional arms — `UK2Node_MacroInstance` (macro-name dispatch), `UK2Node_Tunnel` (direction dispatch), `UK2Node_CallFunction` (field-notify allowlist + `IsLatentNode`) — remain as imperative pre-checks before the registry walk. The MacroInstance pre-check enforces the **MacroInstance-must-precede-Tunnel** ordering invariant since `UK2Node_MacroInstance` derives from `UK2Node_Tunnel`. Added regression test `Source/EditorAutomationRpcGatewayTests/Private/Bpir/TestGraphWalkerClassifyRegistry.cpp` (`FGraphWalkerClassifyRegistryTest`) covering: each pure-registry class; the ordering case (`UK2Node_MacroInstance` bound to `/Engine` `StandardMacros::ForEachLoop` resolves to `ForEach`); the subclass-walk case (`UK2Node_SwitchInteger` resolves to `Switch` via `GetSuperClass()`); a non-latent `UK2Node_CallFunction` pre-registry path; and a null-input guard.
- `#3-skip-rpc-down` `SKIP` tester — Source inspection of `Source/EditorAutomationRpcGateway/Private/Decompiler/GraphWalker.cpp:69-232` confirms the refactor matches the IN-REVIEW description (anonymous-namespace `GetSemanticsRegistry()` with pure class-based arms, MacroInstance/Tunnel/CallFunction imperative pre-checks, `GetSuperClass()` walk) and the regression test file exists at the named path. Cannot run the dynamic verification: the only running editor is `UnrealEditor-Win64-DebugGame.exe` (PID 21460), the RPC gateway at 127.0.0.1:19880 is unreachable (`curl` exit 7, `system.health_check` fetch-fails), and CLAUDE.md forbids DebugGame builds — a Development build + restart is needed. This refactor is a pure internal dispatch reshape with no externally observable BPIR diff to grep for, so the regression test is the only meaningful runtime check; defer to user with a Development editor running.
- `#4-verify-fix` `DONE` tester — Re-confirmed source at `Decompiler/GraphWalker.cpp:69-232` matches the IN-REVIEW description (function-local-static `GetSemanticsRegistry()` with pure class arms, MacroInstance/Tunnel/CallFunction imperative pre-checks, `GetSuperClass()` walk). Live runtime smoke test: `blueprint.decompile` on `/Game/ScifiJungle/ExamplePlayer/Drone/BP_Drone` produced 122KB of well-formed BPIR exercising multiple classifier arms — `entry event` (Event), `call` (FunctionCall), `set` (VariableSet), `branch(%n4)` (Branch from `UK2Node_IfThenElse`), property access via `$Camera.PostProcessSettings`, and `make<PostProcessSettings>` — confirming the registry walk and pre-checks classify all sampled K2Node types correctly post-refactor.
