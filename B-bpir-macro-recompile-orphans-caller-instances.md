---
id: B-bpir-macro-recompile-orphans-caller-instances
title: "compile_bpir on a user macro silently orphans K2Node_MacroInstance nodes in callers"
status: DONE
severity: High
category: bug
tags: [bpir, compiler, macro, silent-failure, append-mode]
---

# Recompiling a macro silently breaks its callers

When `blueprint.compile_bpir` mode=`append` (the default upsert mode)
recompiles a user-defined `entry macro Name(...)` body, the macro graph
is deleted and recreated under a new `UEdGraph` identity. Existing
`UK2Node_MacroInstance` nodes in OTHER graphs that reference the old
macro graph lose their binding and are silently dropped by the orphan
cleanup pass. No warning is emitted; the recompile reports
`success: true`.

The caller graph's exec wiring breaks too: the upstream exec pin that
fed the dropped macro instance is left dangling. Subsequent decompile
of the caller shows the branch's `true ->` (or whatever exec target
pointed at the macro instance) is GONE — the label is missing entirely
because there's no longer a node at that exec target. Callers fail
silently at runtime — the click / event handler just does nothing.

This is the partner side of
`B-bpir-compile-cant-resolve-self-macros`: even if you notice the
breakage, you can't restore the caller via BPIR because the
`macro <SelfMacroName>` syntax doesn't resolve on compile.

## Repro

1. BP `A` has macro `MyMacro` (any body).
2. BP `A` event `Foo` calls `macro MyMacro()`.
3. `blueprint.compile_bpir` with mode=`append` on the macro body
   (e.g. to edit one line of `MyMacro`). Reports `compiled: true`,
   `success: true`, no warnings about orphans.
4. `blueprint.decompile` of `Foo` shows the `macro MyMacro()` call
   is gone. The branch / exec target that pointed at it is dangling
   (e.g. `branch(...) [false -> @else]` with no `true ->` clause).
5. Running the event at runtime — the macro body never executes.

Session evidence: recompiled `W_MissionSelectButton::LoadMission` macro
via `compile_bpir` to add four GI assignments. Mission tiles became
unclickable. Post-mortem decompile of `Button.OnClicked`:
```
@else:
    %n1 = branch($AutoLoadMissionOnClick) [false -> @else_2]
```
The `true ->` exec link and the `@then_2: macro LoadMission(...)`
section that used to exist are GONE. Same pattern repeated on
`W_MyLessonsListItem` after I recompiled its `LoadMission` macro
— `BP_OnClicked` lost its `@isvalid: macro LoadMission(...)` body.

## Impact

- HIGH. Silent runtime breakage of unrelated graphs from a routine
  macro edit. User has no signal anything went wrong until they
  click the button and nothing happens.
- Combines with `B-bpir-compile-cant-resolve-self-macros` to create
  a no-recovery trap: the natural fix (recompile the caller with
  `macro LoadMission(...)` restored) also fails. Forced workaround
  is inlining the macro body into every caller, abandoning the
  macro entirely.
- Mode=`append` is the documented and recommended iteration mode
  for `compile_bpir`. This breakage hits any iterative edit on a
  user macro.

## Workaround

After recompiling a user macro:
1. Decompile every BP in the project that might call this macro.
2. For each caller, inline the macro body into the caller via
   another `compile_bpir`.
3. Delete the (now-unreferenced) macro graph.

Loses round-trip fidelity, bloats callers, and requires knowing
all callers up front.

## Fix

Two complementary changes:

1. **Preserve macro graph identity across upsert.** In Phase 0,
   when the entry being upserted is `entry macro <Name>`, detect
   the existing `UEdGraph` on `Blueprint->MacroGraphs` and re-use
   it instead of deleting + creating fresh. Clear the existing
   body (delete every node reachable from the entry/exit tunnels)
   but keep the `UEdGraph` UObject and the tunnel pair intact, so
   external `UK2Node_MacroInstance` references stay bound.

2. **Warn loudly when references would be orphaned.** Before
   completing a macro recompile, walk
   `FBlueprintEditorUtils::FindAllReferencesToObject(MacroGraph)`
   for the OLD graph (if delete+recreate is still chosen for some
   case). Any `UK2Node_MacroInstance` found in another graph
   should produce a `warnings:` entry listing
   `{ assetPath, graphName, nodeId, macroName }`. If at least
   one is found, the call must not silently succeed — either
   re-bind the instances to the new graph or fail the compile
   with an actionable error.

Add a regression test: BP `A` with macro `M` called from event `E`.
Recompile `M` body. Assert (a) the `K2Node_MacroInstance` in `E`
is still present, (b) its `MacroGraph` pointer is bound to the
post-recompile macro graph, (c) the upstream exec wire into the
macro instance is intact.

## Related

- `B-bpir-compile-cant-resolve-self-macros` — partner bug. Even
  with the orphan warning, you can't restore the caller via BPIR.
- `F-blueprint-add-macro` — same imperative-vs-bulk asymmetry,
  one level higher (graph creation, not graph mutation).

## History
- `#1-initial-repro` `OPEN` reporter — Session: recompiled `W_MissionSelectButton::LoadMission` (added four `set GI.Forced*` assignments) via `compile_bpir`. Reported `compiled: true, success: true, status: "UpToDate"`. After this, mission tiles became unclickable. Decompile of `Button.OnClicked` showed `branch($AutoLoadMissionOnClick) [false -> @else_2]` — `true ->` and the entire `@then_2: %n2 = macro LoadMission(...)` block were silently removed. Same pattern repeated on `W_MyLessonsListItem` after recompiling its `LoadMission` macro (`@isvalid: macro LoadMission(UserFacingExp: %n6)` was dropped from `BP_OnClicked`). Workaround was inlining the macro body into both callers via fresh `compile_bpir`. No warning surfaced during either recompile.
- `#2-preserve-macro-graph-callers` `IN-REVIEW` developer — Preserved replace-mode macro graph identity in `BpirCompiler`, rebuilt reused macro body and tunnel pins without rollback-deleting the graph, reconstructed same-Blueprint `UK2Node_MacroInstance` callers after the signature rebuild, and added `FBpirMacroRecompilePreservesCallerInstancesTest` to assert caller GUID and exec links survive macro recompilation.
- `#3-preserve-caller-pin-state` `IN-REVIEW` developer -- Strengthened macro caller snapshots to cover every existing caller pin, including unlinked/defaultless pins, exact `FText` defaults, pin types, default objects, and links; expanded `FBpirMacroRecompilePreservesCallerInstancesTest` to assert typed/defaulted pins survive successful recompiles and failed signature changes roll back without mutating callers.
- `#4-verify-fix` `DONE` tester — Verified: created temp `W_McpVerifyTemp_bpir_macro_recompile` (UserWidget), compiled BPIR with `entry macro MyMacro()` body `PrintString("v1")` + caller `entry event E()` containing `%n1 = macro MyMacro()`. Recompiled just the macro body (`"v1"` → `"v2"`) via `compile_bpir` mode=append. Post-recompile `blueprint.decompile` still shows `entry override E() { %n0 = macro MyMacro() @(264, 864) }` with the macro call intact and exec wiring preserved, and the macro body shows the new `"v2"` literal. Pre-fix, the caller's `macro MyMacro()` call would have been silently dropped. Temp BP deleted.
