---
id: F-blueprint-add-dispatcher
title: "No way to declare an Event Dispatcher (FMulticastDelegateProperty) on a Blueprint"
status: DONE
severity: High
category: feature
tags: [blueprint, dispatcher, delegate, multicast-delegate, bpir, imperative-api]
---

# No way to declare an Event Dispatcher on a Blueprint

BPIR §2.10 exposes `call_dispatcher`, `bind_dispatcher`, `unbind_dispatcher`,
and `clear_dispatcher`, but every one of them assumes the dispatcher
property (`FMulticastDelegateProperty`) already exists on the target
Blueprint or external class. There is no RPC or BPIR statement that
*creates* a new dispatcher: `blueprint.add_variable` rejects delegate
types (its allow-list is for `FProperty` scalar/struct/object pins, not
`FBPVariableDescription` with the `CPF_MulticastDelegate` flag), and the
only path that produces a dispatcher today is the hardcoded
`interaction.add_interaction_events` helper, which emits a fixed
signature for the interaction subsystem and is not reusable for
arbitrary dispatcher authoring.

Source confirms the gap:
- `Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/`
  contains zero matches for `add_dispatcher` or
  `MulticastDelegateProperty`-as-creation; the existing
  `MulticastDelegateProperty` hits are read-only resolutions inside
  `CreateComponentEventNode` (BlueprintHandlerUtils.cpp:1682).
- `Blueprint->DelegateSignatureGraphs` is enumerated by the graph
  search/dump paths (BlueprintGraphHandler.cpp:848, 3306, 3323) but is
  never appended to by any handler — the array is read-only from the
  RPC surface.
- The generated RPC catalog (`docs/rpc-method-reference.generated.md`)
  contains no `add_dispatcher` / `remove_dispatcher` entries.

**Impact:** Any Blueprint that needs a new dispatcher must be opened
manually in the editor; the entire authoring half of the dispatcher
workflow is unreachable via MCP. BPIR round-trips are one-way: the
decompiler can emit `call_dispatcher OnFoo(...)` for an existing
dispatcher, but a fresh BP cannot be brought to that state from text.
Blocks fully automated authoring of any system that uses dispatchers as
its primary event surface (gameplay events, UI notifications, ability
callbacks, custom subsystem broadcasts).

**Workaround:** Hand-author the dispatcher in the BP editor (Variables
panel → "+" → switch type to a multicast delegate), then drive
`bind/call/unbind/clear` via BPIR. The interaction-events RPC is not
generalizable.

**Proposal:** Add `blueprint.add_dispatcher(path|blueprintPath, name, params?, save?)`.

The handler creates a Blueprint event dispatcher using UE's editor recipe:
add a member variable with `FEdGraphPinType::PinCategory =
UEdGraphSchema_K2::PC_MCDelegate`, create a delegate signature graph named
exactly `<Name>` (not `<Name>__DelegateSignature`), add default nodes and
function terminators, mark the entry editable, apply
`FUNC_BlueprintCallable | FUNC_BlueprintEvent | FUNC_Public`, add requested
params to the `UK2Node_FunctionEntry`, append the graph to
`Blueprint->DelegateSignatureGraphs`, mark structurally modified, compile, and
optionally save. The generated `UFunction` will be
`<Name>__DelegateSignature` after compile.

`blueprint.remove_dispatcher` and `blueprint.set_dispatcher_signature` are out
of scope for this ticket; removal can be handled separately after
reference-safety behavior is specified.

After this ships, the BPIR round-trip closes: a freshly created BP can
be brought to a complete dispatcher state via `add_dispatcher` +
`compile_bpir` with `call_dispatcher` / `bind_dispatcher` statements,
and the decompiler's existing emission of `call_dispatcher` round-trips
against an authoring path of the same primitive.

**Implementation surface:** `FBlueprintEditorUtils::AddMemberVariable` with
`PC_MCDelegate`, a manually created delegate signature graph named exactly
`<Name>`, `Blueprint->DelegateSignatureGraphs.Add(...)`, and the same BPIR
type-to-pin helpers used by existing function-param resolution. Recompile via
the shared compile helper used by other mutating BP handlers.

**Cross-ref:** Pairs with [`F-widget-event-binding`](F-widget-event-binding.md)
(which solved the *consumption* side for widget BndEvt bindings) and
the existing dispatcher-consumer tickets
[`B-bind-dispatcher-self`](B-bind-dispatcher-self.md),
[`B-bind-dispatcher-event-stripped`](B-bind-dispatcher-event-stripped.md),
[`E-bind-dispatcher-redundant-self-wire`](E-bind-dispatcher-redundant-self-wire.md),
[`B-bpir-bind-dispatcher-external-target-local-event`](B-bpir-bind-dispatcher-external-target-local-event.md).

## History
- `#1-no-create-dispatcher-rpc` `OPEN` reporter — BPIR §2.10 ships `call_dispatcher`/`bind_dispatcher`/`unbind_dispatcher`/`clear_dispatcher` but assumes the dispatcher already exists; `blueprint.add_variable` rejects multicast-delegate types and only the hardcoded `interaction.add_interaction_events` produces dispatchers. Confirmed by source sweep: no `add_dispatcher` in `Handlers/Blueprint/`, no `DelegateSignatureGraphs.Add(...)` writes anywhere, no entry in `rpc-method-reference.generated.md`. Proposes `blueprint.add_dispatcher` / `blueprint.remove_dispatcher` / `blueprint.set_dispatcher_signature` to close the authoring half of the BPIR dispatcher round-trip.
- `#2-add-dispatcher-rpc` `IN-REVIEW` developer — Narrowed the ticket to blueprint.add_dispatcher, implemented dispatcher property plus correctly named delegate signature graph, and added regression coverage for property flags, graph naming, and generated signature function.
- `#3-verify-add-dispatcher` `DONE` tester — Verified: created `/Game/App/UI/Test/W_McpVerifyTemp_FBlueprintAddDispatcher`, ran `blueprint.add_dispatcher` for `OnMcpVerified` with `string` and `int` params, observed `compiled=true`, `status=UpToDate`, `signatureGraph=OnMcpVerified`, `signatureFunction=OnMcpVerified__DelegateSignature`, and `blueprint.inspect` showed variable `OnMcpVerified` type `mcdelegate` plus graph `OnMcpVerified`; temp asset deleted with `asset.delete`.
