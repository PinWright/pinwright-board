---
id: B-niagara-create-node-early-return-before-finalize-crash
title: "Editor crash: `niagara.graph.create_node` rejecting an unsupported node class returns between `FGraphNodeCreator::CreateNode` and `Finalize()`, so `~FGraphNodeCreator` fatally asserts — a typed caller error is a hard editor kill"
status: OPEN
severity: Critical
category: bug
tags: [niagara, niagara-graph, create-node, crash, editor-kill, fgraphnodecreator, error-path, multi-agent, shared-editor, test-gap]
encounters: 1
lastSeen: 2026-08-27T19:16:38+05:00
---

# A rejected `nodeClass` takes the whole editor down instead of returning `UNSUPPORTED_NODE_CLASS`

```js
niagara.graph.create_node { nodeClass: "NiagaraNodeEmitter", ... }
```

The handler resolves the class, constructs the node, then rejects it — and dies on the way out:

```
LogPinWrightSubsystem: Warning: Automation request failed (UNSUPPORTED_NODE_CLASS):
  Node class 'NiagaraNodeEmitter' is not supported by niagara.graph.create_node v1.
LogWindows: Error: Assertion failed: bPlaced
  [File:C:\UE_5.8\Engine\Source\Runtime\Engine\Classes\EdGraph\EdGraph.h] [Line: 312]
  Created node was not finalized in a FGraphNodeCreator<EdGraphNode>
LogWindows: Error: [Callstack] UnrealEditor-PinWright.dll!AutoHandler_334_()
  [Handlers/Niagara/NiagaraGraphHandler.cpp:763]
```

The error response is emitted correctly and *then* the process aborts. From the client the call looks
like a transport drop; from the editor it is `appError` → `StaticShutdownAfterError` → exit.

## Root cause

`Handlers/Niagara/NiagaraGraphHandler.cpp:684` opens a stack `FGraphNodeCreator<UEdGraphNode>` and
line 685 creates the node. `Finalize()` is not called until **line 704**. Two `return true` paths sit
in between, and both leave `bPlaced == false`:

| line | path | fatal? |
|------|------|--------|
| 688 | `NODE_CREATE_FAILED` — `CreateNode` returned null | yes |
| 697 | any `ApplyCreateNodePayload` error — `UNSUPPORTED_NODE_CLASS`, `INVALID_OP`, … | yes |

`~FGraphNodeCreator` is an unconditional `checkf(bPlaced, ...)` (`EdGraph.h:312`) — it does not check
whether a node was actually created, so even the null-node path at 688 aborts. Every declared error
this handler can raise after line 684 is an editor kill, not an error.

The class check itself is the last branch of
`NiagaraGraphCreate::ApplyCreateNodePayload` (`NiagaraGraphHandler.cpp:571`), which runs **after** the
node has already been constructed. A class the verb does not support is knowable from `nodeClass`
alone, before any graph mutation.

## Why the test suite is green

`Tests/Niagara/TestNiagaraGraphCreateNode.cpp:124-135` covers exactly this case — and passes. It calls
`NiagaraGraphCreate::ApplyCreateNodePayload(Node, nullptr)` **directly**, on a
`NewObject<UNiagaraNodeAssignment>(GetTransientPackage())`. No `FGraphNodeCreator` is ever
constructed, so the destructor that does the killing is not on the path under test. Same for the
`INVALID_OP` case at :110-121. This is the failure mode `agent-conventions.md` § *Tests + fixtures*
warns about — the test does not call the same production symbol the handler does — and here it hides
a Critical, not a cosmetic.

Test 6 in that file (`:136+`) *does* go through `InvokeHandlerWithCapture`, but only for
`CLASS_NOT_FOUND`, which returns at line 666 — **before** the creator is constructed. The one
dispatcher-level error case exercised is the one case that cannot crash.

## Suggested fix

1. **Validate the node class before constructing anything.** Split the class-support check out of
   `ApplyCreateNodePayload` into a predicate that takes a `UClass*`, and run it next to the
   `ResolveNiagaraSubclassByPath` check at line 662. `UNSUPPORTED_NODE_CLASS` then returns at 669
   with no transaction, no graph `Modify()`, and no half-built node.
2. **Make the remaining window structurally safe.** The payload apply and the null check still sit
   inside the creator's lifetime. Either scope the creator to a block that always finalizes, or wrap
   the early returns so `Finalize()` runs first and the node is removed with
   `Graph->RemoveNode(NewEdNode)` on the error path. A bare `return` between `CreateNode` and
   `Finalize` is never correct with this engine type — worth an audit note next to the other
   `FGraphNodeCreator` call sites (all ~45 others finalize immediately after `CreateNode`; this
   handler is the only one that does work in between).
3. **Move the error-path coverage to the dispatcher seam.** `UNSUPPORTED_NODE_CLASS` and `INVALID_OP`
   need `InvokeHandlerWithCapture` cases against a real `UNiagaraGraph`, the way test 6 does for
   `CLASS_NOT_FOUND`. As written, the unit tests would stay green through the entire fix and through
   a revert of it.

## Impact

Critical. This is a **shared editor** with four agents working in it, and one caller's rejected
argument terminated the process for all of them — the fourth crash in this session. Everything unsaved
in the level and in every open asset editor is lost, and the crash is silent to the caller (the error
response is sent before the abort, so the client sees `UNSUPPORTED_NODE_CLASS` and a dead transport
one call later).

The trigger is not exotic. `B-niagara-authored-emitter-forces-inert` § `#2` names the missing
`UNiagaraNodeEmitter` in the system graph as the root cause of emitters that never simulate, and
`niagara.graph.create_node` is the obvious verb to reach for from that diagnosis — so this ticket's
repro is the natural next step for anyone reading that one. `F-niagara-graph-create-node` is `DONE`,
which advertises the verb as available for "the ~30 other `UNiagaraNode*` subclasses"; the v1
supported list is much shorter, and every class outside it is a landmine.

## Environment

UE 5.8, `EAContentExamples58`, `/Game/Maps/Atlantis`, 2026-08-27T19:16:38+05:00 (log UTC
14:16:38). Log: `Saved/Logs/EAContentExamples58.log`.

## History

- `#1-reported` `OPEN` reporter — Observed as a bystander: another agent in the same editor issued
  the call while I was mid-measurement on the same level. Not reproduced deliberately (doing so
  costs everyone in the editor another crash). Root cause read straight from source:
  `NiagaraGraphHandler.cpp:684/688/697/704` against `EdGraph.h:312`, and the test gap from
  `Tests/Niagara/TestNiagaraGraphCreateNode.cpp:124-135`.
