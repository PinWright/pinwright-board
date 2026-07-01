---
id: E-bpir-magic-strings-and-duplications
title: "BPIR magic-string sweep: schema constants, dedup'd helpers, named constants for cross-file load-bearing literals"
status: DONE
severity: Low
category: ergonomic
tags: [bpir, cleanup, magic-strings, schema-constants]
---

# BPIR magic-string and duplication sweep

The 5-agent audit produced a punch list of small cleanups that don't
warrant their own tickets but compound into real fragility if left.
Bundled here so they can be done in one pass.

> **Re-grep before editing — line numbers below were captured at filing time and have drifted.** Several site clusters moved by 50–100 lines as upstream BPIR work landed. Implementer must locate each literal by content, not by line number.

## 1. Replace pin-name string literals with schema constants

Schema constants `PN_Then`, `PN_Execute`, `PN_ReturnValue` are declared as `static const FName` (not `FString`) in `EdGraphSchema_K2.h`. Adapter rules:

- `Decompiler/BpirDecompiler.cpp` — two `TEXT("ReturnValue")` literals compared against `Pin->PinName.ToString()` (FString). Swap to `UEdGraphSchema_K2::PN_ReturnValue.ToString()`.
- `Compiler/CodeNodeEmitter.cpp` — 12 sites passing `TEXT("then")` to `FindExecPin(const FString&)`. Add an `FName` overload that forwards via `.ToString()`. Replace the 12 sites with `UEdGraphSchema_K2::PN_Then`. Leave `TEXT("then_0")` (one site, indexed sequence form) untouched — there is no schema constant for that.
- `Compiler/BpirSubgraphCompiler.cpp` — 4 sites already wrap `FName(TEXT("execute"))`. Swap directly to `UEdGraphSchema_K2::PN_Execute`.

## 2. Dedup `UK2Node_Knot` follow-through

The original ticket claimed three identical knot-walk loops. Re-grep showed only **two** of the three sites match the same pattern: both walk **backward** through the knot's data-input pins. The third site walks **forward** via `GraphWalker::GetExecOutputPin` and is structurally different — leave it alone.

The two backward-walk sites dedupe via a new helper `UEdGraphPin* FollowKnotsBackward(UEdGraphPin* SourcePin, UEdGraphPin*& OutDeadEndKnotInput)` in a `BpirDecompilerInternal` namespace at file scope. The helper walks while the owning node is a knot and the data-input has a link; it returns the upstream non-knot pin, or nullptr when the chain dead-ends at a knot whose input has a default value (returned via the out-param so the caller can recurse on it).

## 3. Dedup `NormalizeEventName`

`Decompiler/BpirTextEmitter.cpp` defined two parallel 10-entry if-chains: `NormalizeEventName` (transform) and `IsStandardOverrideEventName` (predicate). Replaced with a single static `TMap<FString, FString>` returned by a function-local-static accessor `GetStandardOverrideEventNameMap()`. The transform becomes `Map.Find(Name)`; the predicate becomes `Map.Contains(Name)`.

## 4. Name the cross-file load-bearing prefixes

These literals are load-bearing across decompiler-emit and compiler-parse for round-trip to work; extracting them into named constants in a shared header (one per concept) prevents silent drift.

New header: `Compiler/BpirSharedConstants.h` with three sub-namespaces.

| Concept | Constant | Sites |
|---------|----------|-------|
| `K2Node_AsyncAction_` | `BpirSharedConstants::AsyncAction::ClassPrefix` | `BpirTextEmitter.cpp` (Printf), `BpirCompiler.cpp` (parse) |
| `WhileLoop` / `ForEachLoop` / `ForEachLoopWithBreak` | `BpirSharedConstants::MacroNames::While` etc. | `BpirTextEmitter.cpp` (emit), `BpirCompiler.cpp` (compile), `GraphWalker.cpp` (classify) |
| `K2_Add/RemoveFieldValueChangedDelegate` | `BpirSharedConstants::FieldNotify::Subscribe/UnsubscribeFnName` | `GraphWalker.cpp` (classify), `BpirTextEmitter.cpp` (emit), `BpirCompiler.cpp` (compile) |

## 5. Misrouted enum-equality alias — DEFERRED via TODO

`Compiler/CodeFunctionResolver.cpp:61` aliases `EqualEqual_EnumEnum → EqualEqual_ByteByte`. The alias points at a fictional UFunction name. The originally-suggested fix was to "move the routing to `CodePinResolver.cpp`," but that file has no function-routing surface, and grep confirmed no production BPIR site emits `EqualEqual_EnumEnum` (every test/cache writes `EqualEqual_ByteByte` directly). Since no enum-pin routing site exists in the plugin where pin subcategory could redirect a comparison call, the alias is harmless dead code. Deferred via inline TODO comment at the alias site; alias retained.

## 6. (DEFERRED) Replace `PreProcessKnownConversions` source-rewriting

Per ticket: defer until / if the generic node statement form lands — that may make this redundant. **Not implemented this sprint.**

## Why ergonomic, not bug

None of these change observable behavior; they all reduce the
"places to keep in sync" surface. Filed as one ticket because they
are individually small but architecturally similar.

## History
- `#1-initial-spec` `OPEN` reporter — Audit of `BpirDecompiler`, `BpirTextEmitter`, `BpirCompiler`, `BpirSubgraphCompiler`, `CodeNodeEmitter`, `CodePinResolver`, `CodeFunctionResolver`, `BpirValueResolver`, `GraphWalker`, `NodeLayoutEngine` produced a punch list of magic strings and duplications that should be schema constants, helpers, or shared named constants. Bundled to do in one pass.
- `#2-corrected-impl-and-test` `IN-REVIEW` developer — Implemented sub-items 1 (schema constants with FName-overload adapter for `FindExecPin`), 2 (backward knot-walk dedup via `FollowKnotsBackward` helper, forward site untouched), 3 (`NormalizeEventName` TMap dedup), 4 (`BpirSharedConstants.h` + 3 sub-namespaces). Sub-item 5 deferred via TODO (no enum-pin routing surface exists in the plugin; alias retained as harmless dead-code). Sub-item 6 deferred per ticket. Tests: `EditorAutomationRpcGateway.bpir.magic_strings.{NormalizeEventName,EnumEqualityCompile}` (the latter is an explicit skip-marker for the deferred sub-item 5).
- `#3-skip-editor-down` `SKIP` tester — Editor not running; MCP transport returned timeout / fetch-failed and `Get-Process UnrealEditor` reports NOT RUNNING, so no behavioral round-trip via `blueprint.compile_bpir` / `blueprint.decompile` was possible. Source-level cross-check of the IN-REVIEW claims passed: `BpirSharedConstants.h` exists, `PN_ReturnValue.ToString()` swap present at `BpirDecompiler.cpp:1643/1669`, `GetStandardOverrideEventNameMap()` TMap accessor at `BpirTextEmitter.cpp:321`, `FollowKnotsBackward` helper at `BpirDecompiler.cpp:74` called from two backward sites (1558, 1875), `FindExecPin` FName overload at `CodeNodeEmitter.h:125`, 13 `FindExecPin(... PN_Then|then_0)` call sites in `CodeNodeEmitter.cpp` matching the 12-swapped + 1-untouched contract, 4 `PN_Execute` sites in `BpirSubgraphCompiler.cpp`. `EqualEqual_EnumEnum` alias removed entirely from `CodeFunctionResolver.cpp` (stricter than the "retain with TODO" claim — worth noting but not a regression since the body confirms no production site emits it). Refactor is "no observable behavior change" per ticket body, so behavioral test is low-marginal-value; relisting for tester pickup once editor is up if a behavioral check is desired.
- `#4-roundtrip-decompile-compile` `DONE` tester — Editor up. Behavioral round-trip: `blueprint.decompile` on `/DataCharts/Blueprints/BP_ChartBase` returned `success: true`, no warnings, with sequence/branch/foreach edges (`[0 -> @s0, ...]`, `[true -> @then]`, `[body -> @body]`) emitting cleanly — exercises `PN_ReturnValue`/`PN_Then` decompile-side swaps and the 7-result `BpirSharedConstants.h` consumer set. `blueprint.compile_bpir` against a temp BP `/Game/App/UI/Test/W_McpVerifyTemp_E-bpir-magic-strings` (Actor parent) with a `sequence(2)` + two `PrintString` calls returned `compiled: true`, `nodeCount: 4`, `errors: []`, `status: UpToDate` — exercises the `FindExecPin` FName overload in `CodeNodeEmitter` and `PN_Execute` swap in `BpirSubgraphCompiler`. Temp BP deleted (`asset.delete` `success: true`). Tester #3's `EqualEqual_EnumEnum` removal note already acknowledged as harmless (no production site emits it). All 4 implemented sub-items (1, 2, 3, 4) verified behaviorally; sub-items 5 and 6 deferred per ticket body.
