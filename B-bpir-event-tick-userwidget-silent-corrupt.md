---
id: B-bpir-event-tick-userwidget-silent-corrupt
title: "`compile_bpir entry event Tick` on a UserWidget silently corrupts the existing override-Tick chain and adjacent custom-event bodies"
status: DONE
severity: High
category: bug
tags: [bpir, append-mode, event-name-map, userwidget, silent-corruption, shared-nodes]
---

# `entry event Tick` on UserWidget reuses/phantoms instead of erroring; corrupts adjacent custom events

`FBpirCompiler::SetupEntryPoint` (`Compiler/BpirCompiler.cpp:3457-3466`) dispatches `EBpirEntryKind::Event` for `"Tick"` through `EventNameMap` (`BpirCompiler.cpp:1259`), which rewrites it to `ReceiveTick`. On a `UUserWidget`-parented BP, the parent's tick UFunction is literally named `Tick` (`UMG/Public/Blueprint/UserWidget.h:545-546`), not `ReceiveTick`. `FCodeNodeEmitter::CreateEventNode` (`Compiler/CodeNodeEmitter.cpp:633-639`) calls `FBlueprintEditorUtils::FindOverrideForFunction(BP, EventClass, "ReceiveTick")` — does NOT find the existing override (its `EventReference.MemberName == "Tick"`), then creates a fresh phantom `UK2Node_Event` referencing the nonexistent `ReceiveTick` UFunction. The pre-existing `entry override Tick(...)` survives untouched on the ubergraph (append/Default mode skips Phase 0 deletion entirely, `BpirCompiler.cpp:2040-2062`). The new body emits off the phantom's Then pin.

This is the **silent-corruption sibling** of the loud failure tracked by `B-bpir-event-name-map-class-blind-userwidget-tick` — that ticket's workaround note ("rewrite `entry override Tick` to `entry event Tick(...)`... may contribute to the corruption vector tracked separately") is exactly this vector.

## Symptom

- Compile returns `success: true, status: UpToDate, errors: []`.
- Two `Tick`-shaped event roots now coexist: the original `Tick` override and a phantom `ReceiveTick` event.
- On widgets where the asset author's pre-existing `Tick` body shares K2Nodes with another custom event (see Shared-node note below), decompile after the compile shows the adjacent custom event with an **empty body** and call sites prefixed `SKEL_<Class>::<Event>()`. Subsequent `blueprint.compile` does not repair either symptom.

## Reproduced this session

- `/App/App/UI/W_FoundGasLeaks` — pre-compile dump (`.editor-automation/asset-dumps/App/App/UI/W_FoundGasLeaks/bpir.txt`) shows `entry override Tick` and `entry custom_event OnSearchStateChanged_Event` decompile-rendering identical positions for `GetTotalLeakCount @(1672, 432)`, `Format_Text @(1918, 295)`, `set $ScoreText.Text @(2219, 242)` — strong evidence of shared K2Node instances (positions are unique per node). After `compile_bpir { mode: <default>, code: "entry event Tick(...) {...}" }`, OnSearchStateChanged_Event decompiles empty and the Construct call rewrites as `call SKEL_W_FoundGasLeaks::OnSearchStateChanged_Event()`.
- Same pattern on `/App/App/UI/W_GasLeaksDistance` and `/App/App/UI/W_GasLeaksTargetFound`.

## Hypothesis on mechanism

The proposed entry that filed this bug speculated a Phase-0 cross-kind name match — **that is wrong**: Phase 0 is gated to Replace mode only (`BpirCompiler.cpp:2062`), so append mode performs no deletion. The real mechanism is two-fold:
1. `CreateEventNode` (`CodeNodeEmitter.cpp:633-639`) reuse-by-name fails because EventNameMap mapped `Tick → ReceiveTick` while the existing override's MemberName is the UserWidget-literal `Tick`. A phantom `ReceiveTick` event is created instead of either reusing or erroring.
2. Wiring the new body off the phantom's Then pin, when underlying graph nodes were shared between the original `Tick` body and a custom event (as the dump demonstrates), perturbs the shared-node exec topology in a way the decompiler then renders as an emptied custom-event body. The exact link-break or pin-reuse step that produces the SKEL_ prefix needs traced via a regression test on a planted shared-node shape; current evidence is the post-state, not the mid-compile mutation.

## Fix scope

Root cause is the class-blind `EventNameMap` tracked by `B-bpir-event-name-map-class-blind-userwidget-tick`. Closing that ticket (literal-name lookup against `ParentClass->FindFunctionByName` before the `Receive`-prefix fallback) should make `SetupBuiltinEvent("Tick")` find the existing override on UserWidget instead of phantoming. **Add** to that ticket's fix:
- `CreateEventNode` must error rather than silently create a phantom referencing a UFunction that does not exist on the parent class. A `nullptr` return → handler-level error → atomic rollback.
- Regression: plant a UserWidget with `entry override Tick` whose body shares K2Nodes with a custom event; submit `entry event Tick(...)` in append mode; assert (a) compile fails OR reuses the existing override, (b) the adjacent custom event's body is unchanged, (c) no SKEL_ prefix appears in call sites.

## Workaround

Until the parent fix lands, do not submit `entry event Tick(...)` on UserWidget. Use `compile_bpir mode: extend` against the existing `entry override Tick(MyGeometry, InDeltaTime)` to splice new logic; if the existing override is on a UserWidget and `mode: extend` fails per the parent ticket, edit via `widget.import_xml` + `insert_bpir_at_node` instead. Snapshot affected widgets via git before any session that touches Tick.

## Related

- `B-bpir-event-name-map-class-blind-userwidget-tick` (OPEN) — **root cause**. Its documented workaround triggers this vector.
- `B-bp-saved-state-corruption-mcp-edits` (DONE, reformulated to widget-tree-slot scope) — same silent-success-with-corruption shape but Blueprint-graph rather than widget-tree, and a different mechanism (phantom event from name-map mismatch, not orphan UFunctions / stale CreateDelegates / null panel slots).
- `F-bpir-override-merge-mode` (DONE) — `mode: extend` is the safe path for adding to an existing override.

## History

- `#1-initial-repro` `OPEN` reporter — Filed after handler-source verification of the original proposal's hypothesis. Phase 0 cross-kind deletion claim was incorrect (Phase 0 skipped in append mode); actual mechanism is `EventNameMap` Tick→ReceiveTick rewriting on a UserWidget where the parent UFunction is literally `Tick`, causing `CreateEventNode` to phantom a `ReceiveTick` event instead of reusing the existing override. Pre-compile dump of `W_FoundGasLeaks` confirms shared-K2Node topology between `Tick` override and `OnSearchStateChanged_Event` (identical positions in BPIR decompile output); session evidence reproduced on three widgets. Cross-referenced as the silent-corruption sibling of `B-bpir-event-name-map-class-blind-userwidget-tick`.
- `#2-create-event-node-error` `IN-REVIEW` developer — Closed via the class-aware EventNameMap fix; additionally hardened `CreateEventNode` in `CodeNodeEmitter.cpp` to return null instead of silently creating a phantom event when the parent class has no matching UFunction. Regression test added at `TestBpirUserWidgetTickRoundTrip.cpp`.
- `#3-verified-via-parent-fix` `DONE` tester — Verified indirectly via the parent fix in `B-bpir-event-name-map-class-blind-userwidget-tick #3-verified-userwidget-tick-roundtrips`: the class-aware `ResolveOverrideEventName` now finds the existing UserWidget `Tick` UFunction (the literal name) before falling back to `Receive*`, so `CreateEventNode` no longer phantoms a `ReceiveTick` event on UserWidgets. Same session also confirmed that on three live widgets (W_FoundGasLeaks, W_GasLeaksDistance, W_GasLeaksTargetFound) a fresh BPIR insertion into Tick (via `insert_bpir_before_node`) preserves the original Tick body and adjacent dispatcher event body — no SKEL_ corruption, no emptied custom-event bodies, no phantom event roots. The original silent-corruption symptom did not reproduce. The `CreateEventNode` null-return path is additionally covered by the regression test cited in `#2-create-event-node-error`.
