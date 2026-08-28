---
id: B-niagara-create-node-unfinalized-graph-node-creator-fatal
title: "niagara.graph.create_node kills the editor on every rejected payload: it constructs the node before validating, then returns the typed error without calling NodeCreator.Finalize(), so ~FGraphNodeCreator asserts bPlaced and the process dies"
status: DONE
severity: Critical
category: bug
tags: [niagara, niagara-graph, create-node, editor-crash, assertion, fgraphnodecreator, error-path, returns-clean-then-dies, shared-editor, test-gap]
encounters: 4
lastSeen: 2026-08-28T08:30:00+05:00
---

# `niagara.graph.create_node` takes the whole editor down whenever it rejects the payload

`niagara.graph.create_node` creates the `UEdGraphNode` **before** it validates
class-specific payload, and every validation failure after that point returns
through `Ctx.SendError(...); return true;` without calling `NodeCreator.Finalize()`.
`FGraphNodeCreator`'s destructor then fires

```
Assertion failed: bPlaced [File:C:\UE_5.8\Engine\Source\Runtime\Engine\Classes\EdGraph\EdGraph.h] [Line: 312]
Created node was not finalized in a FGraphNodeCreator<EdGraphNode>
```

which is an `appError`, not a recoverable error, so the editor process terminates.

The client sees a clean, well-formed typed error and nothing else. In the observed
case the `UNSUPPORTED_NODE_CLASS` response was delivered normally and the process
died about **10 seconds later**, so the caller has no reason to associate the two —
the same "returns clean, dies a minute later" shape as
`B-capture-asset-preview-*`/`B-niagara-edit-with-open-asset-editor-slate-crash`.

In a shared editor this kills every other agent's session and every unsaved edit
they hold, from one refused argument.

## Repro (one call, deterministic)

Any node class outside the v1 allowlist. `NiagaraNodeEmitter` is the natural one to
try, because the plugin has no other way to wire an emitter into a system graph
(see `B-niagara-authored-emitter-forces-inert`):

```
niagara.graph.create_node {
  assetPath:  "/Game/PinWrightScratch/NS_TplProbe",
  target:     { kind: "graph", scriptUsage: "SystemSpawnScript" },
  nodeClass:  "NiagaraNodeEmitter",
  x: -400, y: 0, compile: false, save: false
}
```

Response (correct, and delivered):

```
[UNSUPPORTED_NODE_CLASS] Node class 'NiagaraNodeEmitter' is not supported by
niagara.graph.create_node v1.
```

Then the editor dies. The next RPC returns
`EDITOR_NOT_RUNNING: ... (connection refused)`.

Any payload rejection reaches the same destructor, so the allowlist is only the
cheapest trigger, not the only one — `INVALID_OP` from a bad `payload.opName` on a
`NiagaraNodeOp` takes the identical path.

## Evidence

`Saved/Logs/EAContentExamples58.log:7912-7935`, `2026.08.27-14.16.38.848` UTC assert,
`2026.08.27-14.16.48.918` UTC critical-error dump (logs are UTC+0; machine is UTC+5,
so 19:16 local). Callstack innermost first:

```
FDebug::CheckVerifyFailedImpl2()                 AssertionMacros.cpp:797
UnrealEditor-PinWright.dll!AutoHandler_334_()    Handlers/Niagara/NiagaraGraphHandler.cpp:763
FRpcDispatcher::DrainAutoRegistrations'::<lambda_1>::operator()()  Dispatch/RpcDispatcher.cpp:336
FRpcDispatcher::ProcessRequest()                 Dispatch/RpcDispatcher.cpp:646
```

`NiagaraGraphHandler.cpp:763` is the handler's closing `return true;` — the compiler
folds the single stack `FGraphNodeCreator`'s destructor into the function epilogue,
so the frame names the epilogue rather than the early return that skipped
`Finalize()`.

`MassEntityEditorSubsystem::Tick` appears further out only because that is the
tickable that happened to be draining the RPC task graph; it is not involved.

## Guilty source

`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraGraphHandler.cpp`:

```cpp
684:    FGraphNodeCreator<UEdGraphNode> NodeCreator(*Graph);
685:    UEdGraphNode* NewEdNode = NodeCreator.CreateNode(/*bSelectNewNode=*/false, NodeClass);
686:    if (!NewEdNode)
687:    {
688:        Ctx.SendError(TEXT("NODE_CREATE_FAILED"), TEXT("UNiagaraGraph::CreateNode returned null."));
689:        return true;                       // <-- ~FGraphNodeCreator asserts here too
690:    }
...
694:    FNiagaraEditError PayloadError = NiagaraGraphCreate::ApplyCreateNodePayload(NiagaraNode, NodePayload);
695:    if (PayloadError.HasError())
696:    {
697:        Ctx.SendError(*PayloadError.Code, PayloadError.Message);
698:        return true;                       // <-- observed crash path
699:    }
...
704:    NodeCreator.Finalize();                // only reached on the success path
```

`ApplyCreateNodePayload` emits `UNSUPPORTED_NODE_CLASS` at
`NiagaraGraphHandler.cpp:571` for any class outside
`{NiagaraNodeOp, NiagaraNodeInput, NiagaraNodeOutput, NiagaraNodeCustomHlsl,
NiagaraNodeStaticSwitch, NiagaraNodeIf, NiagaraNodeReroute, NiagaraNodeConvert}`.
`ResolveNiagaraSubclassByPath` at `:664` happily resolves **any** `UNiagaraNode`
subclass, so the class check and the class resolution disagree, and every class in
the gap between them is a live grenade.

Engine side, `EdGraph.h:312` is `~FGraphNodeCreator()` -> `check(bPlaced)`.

## What it should do

Validate before constructing. Move the supported-class check (and any payload
validation that can fail) **above** `FGraphNodeCreator NodeCreator(*Graph)` so a
rejected request never creates a node. Belt and braces, the two existing early
returns should call `NodeCreator.Finalize()` (or, better,
`Graph->RemoveNode(NewEdNode)` after finalizing) before returning, so no future
early return can reintroduce the fatal.

A regression test should call `niagara.graph.create_node` with an unsupported
`nodeClass` inside the live loopback seam (`Tests/Infra/TestMcpTransport.cpp`) and
assert the typed error comes back **and the editor is still alive on the next
request**; against current HEAD the test process dies.

Separately worth doing, because it is what sent a caller down this path in the
first place: the `UNSUPPORTED_NODE_CLASS` message should say that
`NiagaraNodeEmitter` in particular cannot be authored here and point at whatever
replaces `niagara.add_emitter`'s missing `RebuildEmitterNodes`
(`B-niagara-authored-emitter-forces-inert`).

## Merged from `B-niagara-create-node-early-return-before-finalize-crash`

Everything in this section came from `B-niagara-create-node-early-return-before-finalize-crash`, an
independent report of this same defect filed 13 s earlier by a third agent in the same editor: same
file and lines (`NiagaraGraphHandler.cpp:684/688/697/704`), same engine assert (`EdGraph.h:312`),
same fix direction. That ticket was **merged into this one and deleted** — nothing below was
discarded, and nothing below duplicates a claim made above.

### Why the test suite is green over the crashing path

`Tests/Niagara/TestNiagaraGraphCreateNode.cpp:124-135` covers exactly this case — and passes. It
calls `NiagaraGraphCreate::ApplyCreateNodePayload(Node, nullptr)` **directly**, on a
`NewObject<UNiagaraNodeAssignment>(GetTransientPackage())`. No `FGraphNodeCreator` is ever
constructed, so the destructor that does the killing is not on the path under test. Same for the
`INVALID_OP` case at `:110-121`.

Test 6 in that file (`:136+`) *does* go through `InvokeHandlerWithCapture`, but only for
`CLASS_NOT_FOUND`, which returns at `:666` — **before** the creator is constructed. The one
dispatcher-level error case exercised is the one case that cannot crash.

This is the failure mode `agent-conventions.md` § *Tests + fixtures* warns about — the test does not
call the same production symbol the handler does — and here it hides a Critical, not a cosmetic. As
written, the unit tests stay green **through the entire fix and through a revert of it**. That is
why `## What it should do` asks for the regression test at the live loopback seam
(`Tests/Infra/TestMcpTransport.cpp`) rather than another direct-call unit test.

### The observed error-then-abort ordering

```
LogPinWrightSubsystem: Warning: Automation request failed (UNSUPPORTED_NODE_CLASS):
  Node class 'NiagaraNodeEmitter' is not supported by niagara.graph.create_node v1.
LogWindows: Error: Assertion failed: bPlaced
  [File:C:\UE_5.8\Engine\Source\Runtime\Engine\Classes\EdGraph\EdGraph.h] [Line: 312]
  Created node was not finalized in a FGraphNodeCreator<EdGraphNode>
LogWindows: Error: [Callstack] UnrealEditor-PinWright.dll!AutoHandler_334_()
  [Handlers/Niagara/NiagaraGraphHandler.cpp:763]
```

The error response is emitted correctly and *then* the process aborts (`appError` ->
`StaticShutdownAfterError` -> exit), which is why the client sees a well-formed
`UNSUPPORTED_NODE_CLASS` and a dead transport on the next call rather than anything connecting the
two.

Both early returns are fatal, not just the observed one:

| line | path | fatal? |
|------|------|--------|
| 688 | `NODE_CREATE_FAILED` — `CreateNode` returned null | yes |
| 697 | any `ApplyCreateNodePayload` error — `UNSUPPORTED_NODE_CLASS`, `INVALID_OP`, … | yes |

`~FGraphNodeCreator` is an unconditional `checkf(bPlaced, ...)`; it does **not** check whether a node
was actually created, so even the null-node path at `:688` aborts.

### Fix detail: where the class predicate belongs

The class check is the last branch of `NiagaraGraphCreate::ApplyCreateNodePayload`
(`NiagaraGraphHandler.cpp:571`), which runs **after** the node has already been constructed. A class
the verb does not support is knowable from `nodeClass` alone, before any graph mutation. Split that
check out into a predicate taking a `UClass*` and run it next to the `ResolveNiagaraSubclassByPath`
check at `:662`; `UNSUPPORTED_NODE_CLASS` then returns at `:669` with no transaction, no graph
`Modify()` and no half-built node.

### This is the only `FGraphNodeCreator` call site that does work before `Finalize()`

All ~45 other `FGraphNodeCreator` call sites in the plugin finalize immediately after `CreateNode`;
this handler is the only one that does work in between. A bare `return` between `CreateNode` and
`Finalize()` is never correct with this engine type — worth an audit note alongside the other call
sites so the pattern is not reintroduced.

### Why callers keep reaching for this verb

`F-niagara-graph-create-node` is `DONE`, and advertises the verb as available for "the ~30 other
`UNiagaraNode*` subclasses". The v1 supported list is much shorter, so every class in the gap is a
landmine. Combined with `B-niagara-authored-emitter-forces-inert` § `#2` — which names the missing
`UNiagaraNodeEmitter` in the system graph as the root cause of emitters that never simulate — this
verb is the obvious next call for anyone reading that diagnosis, which is exactly how all three
reports arrived within ten minutes.

## Impact

Critical. One refused argument on a read-shaped exploratory call terminates the
editor for every agent sharing it, with the silent loss of all unsaved work — this
session lost a live editor shared by several placement agents. It is worse than the
other editor-killers already on the board because there is no unusual sequence
involved: it is a **single call with a wrong argument value**, which is exactly what
discovery looks like, and the verb's own documentation lists the supported classes
in a way that invites trying an unlisted one.

## Distinct from related tickets

- `B-niagara-create-op-bare-leaf-pinless` (IN-REVIEW) is the *success* path on
  `NiagaraNodeOp` producing a pinless `Unknown` node. That fix canonicalizes
  `opName`; it does not touch the early-return-before-`Finalize` fault, and in fact
  a stricter `INVALID_OP` rejection routes through this same fatal.
- `B-niagara-edit-with-open-asset-editor-slate-crash` and the
  `capture_asset_preview` crashes are Slate/teardown faults needing a specific prior
  sequence. This one needs one call and no prior state.
- `E-niagara-create-node-input-type-unverifiable` and
  `E-create-node-operator-symbol-discovery` are ergonomics on the same verb.

severity rationale: impact=editor-killing `appError` with silent loss of every sharing agent's unsaved work, reached from a well-formed call that returns a correct typed error first x reach=any rejected payload on `niagara.graph.create_node`, i.e. the ordinary discovery path for the verb -> Critical.

## History
- `#1-initial-repro` `OPEN` reporter — 2026-08-27, UE 5.8, Atlantis map build (map as forcing function, host `CLAUDE.md` § "What this project is for"). Hit while probing whether `niagara.graph.create_node` could author the `UNiagaraNodeEmitter` that `niagara.add_emitter` fails to create (`B-niagara-authored-emitter-forces-inert` #2). Single call `niagara.graph.create_node {assetPath:"/Game/PinWrightScratch/NS_TplProbe", target:{kind:"graph", scriptUsage:"SystemSpawnScript"}, nodeClass:"NiagaraNodeEmitter", x:-400, y:0}` returned the correct typed `UNSUPPORTED_NODE_CLASS` error, and the editor died ~10 s later on `Assertion failed: bPlaced [EdGraph.h:312] Created node was not finalized in a FGraphNodeCreator<EdGraphNode>` (`Saved/Logs/EAContentExamples58.log:7912-7935`, assert `2026.08.27-14.16.38.848` UTC, crash dump `14.16.48.918` UTC), callstack `FDebug::CheckVerifyFailedImpl2` <- `NiagaraGraphHandler.cpp:763` <- `RpcDispatcher.cpp:336` <- `RpcDispatcher.cpp:646`. Root cause read from source in this checkout: the node is constructed at `NiagaraGraphHandler.cpp:684-685` before `ApplyCreateNodePayload` validates at `:694`, and the rejection path `:697-698` returns without the `NodeCreator.Finalize()` at `:704`, so the stack `FGraphNodeCreator` destructor asserts in the function epilogue (hence the `:763` frame). `NODE_CREATE_FAILED` at `:688-689` is the same shape. The class allowlist lives in `ApplyCreateNodePayload` (`:571`) while `ResolveNiagaraSubclassByPath` (`:664`) accepts any `UNiagaraNode` subclass, so every class in that gap is fatal. Fix direction: validate before constructing. Not verified by re-running the crash — deliberately, since it kills a shared editor.

- `#2-bystander-encounter` `OPEN` reporter — 2026-08-27T19:16:48+05:00, same editor instance, same
  assert. I was the Sequencer/cine agent, not the caller. Adds two things #1 does not have:

  **(a) The cost is not only "unsaved work".** The crash landed between my `sequencer.create`
  (which saves the `.uasset` itself) and the `sequencer.set_display_rate` /
  `sequencer.set_properties` calls that followed it. Those two verbs return
  `pendingSave: true` / `applied: true` and mutate the loaded `UMovieScene` **in memory only**, so
  `/Game/Atlantis/Cine/LS_Atlantis_Flythrough.uasset` survived on disk at its 19:03 mtime with
  the default 30 fps display rate and no playback range. The asset therefore exists, is loadable,
  and is silently wrong — the failure mode this host's `CLAUDE.md` § "Verify a write against disk"
  is written about. Any verb whose response says `pendingSave` is unrecoverable across this crash,
  and a caller that trusted the success response has no signal that it was rolled back.

  **(b) Duplicate pair on the board.** `B-niagara-create-node-early-return-before-finalize-crash`
  (committed 19:20:27, 13 s after this one at 19:20:14) is the same defect from the same assert,
  filed independently by a third agent in the same editor: same file/lines
  (`NiagaraGraphHandler.cpp:684/688/697/704`), same engine assert (`EdGraph.h:312`), same fix
  direction. It carries one thing this ticket does not — the test-gap analysis at
  `Tests/Niagara/TestNiagaraGraphCreateNode.cpp:124-135`, showing why the suite is green over the
  crashing path. Merge that section in here and close the other as a duplicate, or vice versa; do
  not fix twice. *(Resolved on consolidation: that ticket was merged into this one and its
  file deleted — see `## Merged from ...` and `#3-merged-independent-report`.)* Three independent reports inside ten minutes is itself the severity evidence:
  every agent that reads `B-niagara-authored-emitter-forces-inert` reaches for this verb next.

  No new repro run — deliberately, per #1.

- `#3-merged-independent-report` `OPEN` reporter — 2026-08-27T19:16:38+05:00 (log UTC 14:16:38),
  same editor instance, same assert. Originally filed as its own ticket
  `B-niagara-create-node-early-return-before-finalize-crash` (committed 19:20:27, 13 s before this
  one); **merged into this ticket and that file deleted** during a duplicate-consolidation pass —
  its content is preserved in `## Merged from ...` above, not discarded. Observed as a bystander:
  another agent in the same editor issued the call while this reporter was mid-measurement on the
  same level. Not reproduced deliberately (doing so costs everyone in the editor another crash).
  Root cause read straight from source: `NiagaraGraphHandler.cpp:684/688/697/704` against
  `EdGraph.h:312`; test gap read from `Tests/Niagara/TestNiagaraGraphCreateNode.cpp:124-135`.
  Environment: UE 5.8, `EAContentExamples58`, `/Game/Maps/Atlantis`. The log this crash is in was
  rotated by the crash itself and is retained as
  `Saved/Logs/EAContentExamples58-backup-2026.08.27-14.16.48.log` (the original ticket cited the
  live `Saved/Logs/EAContentExamples58.log`, which has since rolled past it).

- `#4-validate-before-construct` `IN-REVIEW` developer — Fixed in
  `Source/PinWright/Private/Handlers/Niagara/NiagaraGraphHandler.cpp`. The v1 supported-class list
  moved out of the tail of `ApplyCreateNodePayload` into a new
  `NiagaraGraphCreate::ValidateCreateNodeClass(UClass*)` (declared in
  `Handlers/Niagara/NiagaraGraphCreateNodePayload.h`), which the handler runs immediately after the
  `CLASS_NOT_FOUND` gate — before the `FScopedTransaction`, before `Graph->Modify()`, and before any
  `FGraphNodeCreator` exists — so `UNSUPPORTED_NODE_CLASS` no longer constructs anything.
  `ApplyCreateNodePayload` delegates its tail to that same predicate, so the allowlist has exactly
  one definition. The remaining window is closed structurally rather than by discipline: the
  creator is scoped to a block holding only `CreateNode()` → `ApplyCreateNodePayload` → node
  position → `Finalize()`, with no `return` inside it; a payload rejection (`INVALID_OP`) is carried
  out in a local and handled after the block, where the finalized node is dropped with
  `Graph->RemoveNode`. The `NODE_CREATE_FAILED` null check at the old `:688` is gone —
  `UEdGraph::CreateNode` routes through `NewObject`, which cannot return null for a concrete class,
  and a null could not have been reported from inside the creator anyway since `Finalize()`
  dereferences the node. That code is now emitted *before* construction for a class carrying
  `CLASS_Abstract | CLASS_Deprecated | CLASS_NewerVersionExists`, which is the case `NewObject`
  would have fataled on. Regression test
  `PinWright.niagara.graph.create_node_guard.RejectionLeavesGraphUnchanged`
  (`Source/PinWright/Private/Tests/Niagara/TestNiagaraGraphCreateNodeCreatorGuard.cpp`) drives the
  verb through the dispatcher against a real `UNiagaraGraph` on a synthetic transient system and
  asserts the guard, not the crash: an unsupported `nodeClass` returns `UNSUPPORTED_NODE_CLASS` with
  the graph node count unchanged; `NiagaraNodeOp` with an unknown `opName` returns `INVALID_OP` with
  the node count unchanged (that is the path that still runs inside the creator's lifetime, so a
  class-check-only fix still fails it); then a supported class succeeds with exactly one node added
  and a non-empty `nodeId`, proving the verb is still reachable and that `Finalize()` still runs.
  Not compiled or run here — the orchestrator owns builds. Closes both reports: the duplicate
  `B-niagara-create-node-early-return-before-finalize-crash` was merged into this ticket and deleted
  (see `#3`) before this fix landed, so there is no second file to flip.

- `#5-verified-fixed` `DONE` verifier — 2026-08-28. Plugin rebuilt from a clean tree at `b79ba53e` and verified against disk, not against the build's own success message: `UnrealEditor-PinWright.dll` 39,898,624 -> 40,644,096 bytes at 2026-08-28 08:11:48, `UnrealEditor-PinWrightGeometry.dll` 4,983,296 -> 5,113,344, canonical link with no `-000N` artifacts in `UnrealEditor.modules`. Editor restarted on that DLL and the ticket's own repro re-run. **Fixed, and the only ticket in this batch with a proven before/after crash.** Previous encounters deliberately never re-ran the repro because it kills a shared editor; this pass had the editor to itself and ran it on both DLLs. Before: the ticket's verbatim call on `/Game/PinWrightScratch/NS_TplProbe` returned the correct `[UNSUPPORTED_NODE_CLASS]` and killed editor pid 80620 within 12 s - `Assertion failed: bPlaced [EdGraph.h:312]` at `2026.08.28-03.05.31` UTC in `Saved/Logs/EAContentExamples58.log`, the same assert as #1. After the rebuild the identical call returns the identical typed error and the editor survives. Also exercised the path the class-check hoist does NOT cover, i.e. a rejection raised *inside* the `FGraphNodeCreator` lifetime: `nodeClass:"NiagaraNodeOp"` with `payload.opName:"ThisOpDoesNotExist"` -> `[INVALID_OP] Unknown Niagara op`, editor alive, zero asserts in the fresh log. Both declared error shapes are now survivable. Residual, unrelated to the crash: the refused call still leaves `NS_TplProbe` dirty - see `B-niagara-refused-edit-dirties-package`.
