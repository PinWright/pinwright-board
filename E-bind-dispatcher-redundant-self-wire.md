---
id: E-bind-dispatcher-redundant-self-wire
title: "`bind_dispatcher` emits redundant K2Node_Self wire on CreateDelegate"
status: DONE
severity: Low
category: ergonomic
tags: []
---

# `bind_dispatcher` emits redundant K2Node_Self wire on CreateDelegate

When BPIR emits a `bind_dispatcher <Name>(event: @Handler)` for a same-class handler (handler lives on the same BP as the dispatcher owner), the compiler produces three nodes: `K2Node_AddDelegate`, `K2Node_CreateDelegate`, **and** a `K2Node_Self`, with explicit wires `AddDelegate.Target ← K2Node_Self` and `CreateDelegate.self ← K2Node_Self`.

UE treats both `AddDelegate.Target` and `CreateDelegate.self` as implicit self when left unconnected. The explicit `K2Node_Self` + wires add zero runtime value — just visual clutter on the graph. Decompile shows this as `call Create_Event(self: self)`, which is obvious noise on re-read.

**Impact:**
Every same-class delegate bind emits 1 extra `K2Node_Self` node + 2 extra pin connections. On a BP with 5–10 binds this becomes several dozen extra nodes that a human reader must skip past.

**Workaround:**
None on the authoring side. Users can manually delete the `K2Node_Self` + its wires post-compile; UE auto-routes to self.

**Background:**
The `B-bind-dispatcher-self` DONE fix explicitly wires `CreateDelegate.self` to `K2Node_Self`, but its comment acknowledges "(or leave unconnected)" as equivalent. The fix was motivated by an incorrect wire (self was being wired to Target before), so any explicit wire was an improvement. Now that the fix is in place, the *right* default is probably to leave these pins unconnected and let UE's implicit-self rule handle them.

**Proposal:**
When the handler event lives on the same class as the `bind_dispatcher` emission site (the common case: `event: @HandlerOnSelf`), skip emitting the `K2Node_Self` and leave both `AddDelegate.Target` and `CreateDelegate.self` unconnected. Only emit the `K2Node_Self` (or external-target wire) when `target:` is explicitly specified **and** resolves to a non-self object reference.

## History
- `#1-redundant-self-wire-cosmetic` `OPEN` reporter — Noticed during replay subtask #03 BP authoring. Every `bind_dispatcher` on `W_MyReplaySelect` (3 of them: `OnStateChanged`, `OnOpenReplayRequested`, `HandleStateChanged`) emitted the redundant `Create_Event(self: self)` wire visible in the decompile. Runtime behavior is correct — this is strictly a graph-cleanliness issue. Filing as Low severity since it's cosmetic.
- `#2-guarded-self-wire-emission` `IN-REVIEW` developer — Guarded the CreateDelegate.self wiring in `BpirCompiler.cpp` BindDispatcher opcode (`CreateDelegateNode` lambda, around former lines 3971-3981) with `if (!bSelfContext)`. For same-class binds the pin is now left unlinked — UE's `UK2Node_CreateDelegate::GetScopeClass()` resolves an unlinked self pin to the current BP's class, identical to wiring to a `K2Node_Self`. External-target binds still wire CreateDelegate.self to self (preserves the B-bind-dispatcher-self fix). AddDelegate.Target was already correctly left unlinked for the self-context path. Added regression test `FCompilerIntegrationBindDispatcherSelfContextNoRedundantSelfTest` in `TestCompilerDispatchersOps.cpp` asserting no `K2Node_Self` is emitted, both `CreateDelegate.ObjectInPin` and `AddDelegate.Target` are unlinked, and `CreateDelegate.SelectedFunctionName` still resolves to the handler via implicit-self scope. Existing `FCompilerIntegrationBindDispatcherExternalTargetUsesSelfTest` acts as regression anchor for the external-target path.
- `#3-verified-no-redundant-self` `DONE` tester — Verified on `W_McpVerify_Replay03`: `bind_dispatcher OnVisibilityChanged(event: @HandleOnSelf)` (same-class bind) compiled clean. Inspected resulting nodes — `K2Node_CreateDelegate` has `self` pin with no `linkedTo`, `K2Node_AddDelegate` has `self` pin (subtype `W_McpVerify_Replay03_C`) with no `linkedTo`. The only `K2Node_Self` in the graph is wired to an unrelated explicit `InWorldContextObject: self` argument on a `ShowConfirmationYesNo` call. Decompile confirms: emits `%n0 = call Create_Event()` (no `self: self` arg) where prior to the fix it emitted `call Create_Event(self: self)`. External-target binds not re-tested here but the preserved `B-bind-dispatcher-self` regression test covers that path.
