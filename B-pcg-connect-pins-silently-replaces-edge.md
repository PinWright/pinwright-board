---
id: B-pcg-connect-pins-silently-replaces-edge
title: "pcg.connect_pins reports a hardcoded connected:true and never says that wiring a single-connection pin destroyed the edge already on it — the bit that would say so is computed by the engine, returned from AddLabeledEdge, and discarded twice on the way to the response"
status: IN-REVIEW
severity: High
category: bug
tags: [pcg, connect-pins, silent-data-loss, false-success, hardcoded-response, edge, single-connection-pin, procedural-vegetation, undo, response-shape]
encounters: 1
lastSeen: 2026-08-29T17:00:00+05:00
---

# The engine hands PinWright the answer and PinWright drops it

`pcg.connect_pins` wires a source output pin to a target input pin. When the target pin does not
allow multiple connections — which is **every** Procedural Vegetation node's `In` pin — the engine
breaks whatever was already wired there. The response does not mention it, and cannot, because the
only field that could is a literal.

## Measured

On `/Game/PinWrightScratch/PVTest/PCG_PwVerdictSubclass`, a `ProceduralVegetationGraph` holding one
`UPVSeedGeneratorSettings` node and one stock `PCGCreatePointsSettings` node.

| step | call | response | `pcg.inspect` `edges[]` afterwards |
|---|---|---|---|
| 1 | `connect_pins {DefaultInputNode.In → SeedGenerator_0.In}` | `{"connected":true, …}` | `[{DefaultInputNode.In → SeedGenerator_0.In}]` |
| 2 | `connect_pins {CreatePoints_0.Out → SeedGenerator_0.In}` | `{"connected":true, …}` | `[{CreatePoints_0.Out → SeedGenerator_0.In}]` |

The step-1 edge is **gone**. No warning, no `replacedEdges`, no `warnings[]`, nothing in the payload
that differs from a connect that displaced nothing.

**The obvious client-side defence does not work either.** `edges[]` went from length 1 to length 1.
A caller who wires a graph and checks `pcg.inspect`'s edge *count* after each call sees a perfectly
stable graph while its wiring is being overwritten. Only comparing the edge *set* — from-node,
from-pin, to-node, to-pin, per entry — reveals it, and nothing in the docs suggests you need to.

## Mechanism — three links, and the bit survives the first two

**1. The engine computes it and returns it.** `UPCGGraph::AddLabeledEdge`,
`C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Private/PCGGraph.cpp:1457-1508`:

```cpp
FromPin->AddEdgeTo(ToPin, &TouchedNodes);                                  // :1488
bool bToPinBrokeOtherEdges = false;                                        // :1490
if (!ToPin->AllowsMultipleConnections())                                   // :1493
{
    bToPinBrokeOtherEdges = ToPin->BreakAllIncompatibleEdges(&TouchedNodes);  // :1495
}
...
return bToPinBrokeOtherEdges;                                              // :1507
```

The header states the contract in as many words — `PCGGraph.h:504`: *"Returns true if the To node has
removed other edges (happens with single pins)"*.

**2. `UPCGGraph::AddEdge` throws it away.** `PCGGraph.cpp:1451-1455`:

```cpp
UPCGNode* UPCGGraph::AddEdge(UPCGNode* From, const FName& FromPinLabel, UPCGNode* To, const FName& ToPinLabel)
{
    AddLabeledEdge(From, FromPinLabel, To, ToPinLabel);                    // :1453 — return discarded
    return To;
}
```

`AddEdge` is the Blueprint-exposed convenience wrapper (`PCGGraph.h:476-477`); `AddLabeledEdge` is
C++-only (`PCGGraph.h:505`, no `UFUNCTION`). `PinWrightPCG` links the PCG module natively, so that
distinction costs the fix nothing.

**3. PinWright calls the wrapper and then writes a literal.**
`Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphAuthoring.cpp`:

```cpp
Graph->AddEdge(FromNode, FromPinFName, ToNode, ToPinFName);   // :151 — return discarded again
Graph->MarkPackageDirty();                                    // :152
...
Result->SetBoolField(TEXT("connected"), true);                // :155 — literal
```

`:156-159` set `fromNode`, `fromPin`, `toNode`, `toPin`, all echoes of the request. That is the
complete response: five fields, four of them the caller's own input and one of them a constant.
Nothing between `:151` and `:155` re-reads the graph.

## Why the discarded bool is unambiguous *at this call site*

Worth stating, because `AddLabeledEdge`'s return is overloaded in general: it also returns `false`
from three early failures — null nodes (`:1462`), missing `FromPin` (`:1470`), missing `ToPin`
(`:1478`) — so in the abstract `false` means "broke nothing **or** built nothing".

At this call site it cannot. The handler has already proved both pins exist before it calls, at
`PCGGraphAuthoring.cpp:142-149`, via `FindPinByLabel(FromNode->GetOutputPins(), …)` and
`FindPinByLabel(ToNode->GetInputPins(), …)`, answering `PIN_NOT_FOUND` otherwise; and `FromNode` /
`ToNode` are non-null by `:132-138`. So all three failure paths are unreachable here and the
returned bool carries exactly one meaning: **did this call destroy an existing edge.** The one thing
the return value means is the one thing the response omits.

## Which edge dies is deterministic, and always the caller's earlier work

`UPCGPin::BreakAllIncompatibleEdges` (`C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Private/PCGPin.cpp:283-369`)
walks `Edges` **backwards** (`:307`) and computes `bRemoveEdge = (!AllowsMultipleConnections() &&
bHasAValidEdge)` (`:316`), where `bHasAValidEdge` only becomes true after one edge has been kept
(`:364`). Because `AddLabeledEdge` appends the new edge (`PCGGraph.cpp:1488`) *before* calling it
(`:1495`), the **new** edge is the survivor and every pre-existing connection on that pin is the
casualty. So this is not a race or an ordering accident: incremental authoring destroys earlier
authoring, reliably, in the direction that makes it hardest to notice.

`UPCGPin::AllowsMultipleConnections` (`PCGPin.cpp:444-448`) returns true unconditionally for output
pins, so only the `To` side can ever trigger this — consistent with the branch above, and it means
`fromPin` is never at risk.

## Blast radius: every Procedural Vegetation node

`UPVBaseSettings::InputPinProperties` sets the family's default `In` pin to single-connection —
`C:/UE_5.8/Engine/Plugins/Experimental/ProceduralVegetationEditor/Source/ProceduralVegetation/Private/Nodes/PVBaseSettings.cpp:34`
(function `:28-38`, declared `Public/Nodes/PVBaseSettings.h:49`):

```cpp
FPCGPinProperties& Pin = Properties.Emplace_GetRef(PCGPinConstants::DefaultInputLabel, GetInputPinTypeIdentifier());
Pin.SetRequiredPin();
Pin.SetAllowMultipleConnections(false);            // :34
```

A whole-plugin sweep of `SetAllowMultipleConnections` (37 hits) finds exactly **two** sites that
create a `PCGPinConstants::DefaultInputLabel` pin, and both set it `false`: the one above, and
`UPVGrowerSettings::InputPinProperties` at `PVGrowerSettings.cpp:313`, which does **not** call
`Super::InputPinProperties()` and independently re-enforces the same rule. Every
`SetAllowMultipleConnections(true)` in the plugin is on a named auxiliary pin or an output pin
(`PVBaseSettings.cpp:45`, `PVGrowerSettings.cpp:318`/`:327`/`:346`/`:351`/`:360`/`:372`). So there
is no PV node whose `In` pin tolerates a second connection.

A PV plant graph is 15-40 nodes. Building one through PinWright means calling `connect_pins` dozens
of times into pins that all behave this way, with a response that is byte-identical whether the call
added a wire or replaced one.

## The displaced edge is probably not recoverable by undo — source-only

`BreakAllIncompatibleEdges` does call `Modify()` on both pins before unlinking (`PCGPin.cpp:342`,
`:352`), so the engine side is undo-*ready*. But `Modify()` only snapshots into an **active**
transaction, and this handler opens none: `PCGGraphAuthoring.cpp` contains **zero**
`FScopedTransaction` occurrences, and the whole `PinWrightPCG` module contains exactly one
(`PCGGenerateHandler.cpp:156`, on `pcg.generate`). So there is no transaction for those `Modify()`
calls to record into and `GEditor->UndoTransaction()` should have nothing to restore.

**Marked source-only: undo was not exercised.** It is the single measurement that would move this
ticket to Critical, and it is cheap to take — wire A→C, wire B→C, undo, inspect. Whoever picks this
up should take it first.

## Fix

Swap the wrapper for the one that tells the truth, and publish what it says:

```cpp
const bool bReplaced = Graph->AddLabeledEdge(FromNode, FromPinFName, ToNode, ToPinFName);
...
Result->SetBoolField(TEXT("replacedExistingEdge"), bReplaced);
```

One call changed, one field added, no new computation, no new lookup — and `connected` becomes
defensible as a literal precisely because the case it was hiding now has its own field. A richer
version is available at the same site for free: the handler holds `ToNode->GetInputPin(ToPinFName)`
and could snapshot that pin's `Edges` before the call and name the displaced ones in a
`replacedEdges[]` array, which is what a caller actually needs in order to re-wire. The bool is the
minimum that stops the response from being untrue; the array is what makes the loss repairable.

Whichever is chosen, `Docs/wiki-src/pcg.md:15` — the `connect_pins` bullet, currently about endpoint
resolution only — should say that wiring a single-connection input pin replaces what is on it, and
that PV nodes are all single-connection.

**Workaround until then:** `pcg.inspect` before and after every `connect_pins` and diff the edge
**set**. The count is useless (measured 1 → 1).

## Same shape as

The session's recurring class, stated on `B-foliage-paint-does-no-ground-projection` § *Same shape
as*: *the call succeeds, every number it reports is correct, and the output is wrong because the
deciding number was never reported.* This is the cheapest member of the class yet found — the
deciding bit is not merely uncomputed, it is **computed by the engine, returned across the API
boundary, and discarded twice** (once by `AddEdge`, once by the handler) before a literal is written
in its place.

Nearest siblings, same class, different mechanisms:

- `B-pcg-generate-instancecount-blind-to-species` (OPEN, High) — same namespace, same response
  family; there the number reported is correct and the documentation nominates it as the wrong
  thing to trust.
- `B-component-mesh-swap-silently-unseats-instances` (OPEN, Medium) — the variant where the write
  path states the deciding quantity in a comment and acts on it, and still does not report it.
- `B-foliage-remove-empties-ledger-not-component` (OPEN, Critical) — the variant where the deciding
  number is unreadable through any verb in the namespace.

## Related, and why this one is load-bearing rather than cosmetic

- **`F-pcg-disconnect-pins`** (OPEN, filed this session) — PinWright has no verb that removes an
  edge while leaving both nodes standing. That is what turns this defect from "an incomplete
  receipt" into "your earlier wire is gone and re-creating it is not a one-call operation": the
  repair for a displaced edge on a *multi*-connection pin has no typed route at all. Read the two
  together.
- **`B-pcg-generate-blind-to-non-point-data`** (OPEN, filed this session) — the far end of the same
  session's PV work: having silently mis-wired the graph, nothing `pcg.generate` returns could tell
  you. The two defects compound, which is the argument for fixing the cheap one.
- `B-pcg-inspect-edge-direction-reversed` — the other `pcg` edge-reporting defect. Different verb,
  different direction of error; checked and distinct.

## Noted here, deliberately NOT filed

`PCGGraphAuthoring.cpp` wraps **no** mutation in an `FScopedTransaction` — not `add_node`, not
`connect_pins`, not `remove_node` — against `agent-conventions.md`'s *"Wrap EVERY mutation in
`FScopedTransaction`"*. That is a namespace-wide gap deserving its own ticket rather than a
`connect_pins` defect, exactly as the identical finding on `FoliageHandler.cpp` was handled on
`B-foliage-paint-does-no-ground-projection`. It is recorded here because it is what makes *this*
defect's loss plausibly permanent (see the undo section), not because this ticket should absorb it.

## Dedup

Board-wide search for `connect_pins`, `AddLabeledEdge`, `replacedEdges`, replaced-edge and
single-connection-pin tickets. The `pcg.*` files touching edges are
`B-pcg-inspect-edge-direction-reversed` (read path, direction), `E-pcg-add-node-echo-pin-labels`
(response shape on the node-create verbs), `F-pcg-core-graph` (whose `#3` returned an unrelated
`NODE_NOT_FOUND` resolver bug in this same handler, since fixed), and `F-pcg-authoring-parity`
(graph user parameters). The `niagara.*` namespace has its own `connect_pins`
(`B-niagara-connect-pins-sends-no-graph-notification`, `E-niagara-connect-pins-add-pin-opaque`) —
different handler, different engine API, no shared code. **Nothing on the board mentions a replaced
or destroyed edge, on any namespace.**

## Severity

**High.** Impact class is the rubric's High band verbatim — *silent false-success / silent wrong data
on a normal path, where the caller trusts a result and builds on it.* The response says `connected:
true`, which is true, and says nothing else, and the caller proceeds to wire the next node on a
graph that no longer contains the wire they made two calls ago.

**Critical argued and declined, with the argument named so a reviewer can take it up.** The Critical
band is *"a write that corrupts or loses asset data"*, and this write does lose authored data from a
saved asset. It is declined because the *write* is correct: replacing the occupant of a
single-connection pin is the engine's documented semantic (`PCGGraph.h:504`) and is exactly what the
PCG Graph editor does on the same action — where the user watches the old wire vanish. What is
defective is that PinWright is the surface on which it happens invisibly. The rubric's Critical band
is for a write whose *effect* is wrong; here the effect is right and the *report* is wrong, which is
the High band as written. Two things would flip it: a measurement showing the displaced edge is
unrecoverable by undo (see the source-only section above — this is the one open question), or a case
where the destroyed edge is not re-creatable by a subsequent `connect_pins`.

**Reach modifier declined in both directions, and named.** No bump down: `connect_pins` is one of
`pcg`'s 14 verbs and the one a graph author calls most often — a 40-node PV graph calls it dozens of
times — so it is nothing like a rare edge path. No bump up: PCG graph authoring is not an
every-session activity across the plugin's whole surface, and a bump up from High lands on Critical,
which the paragraph above declines on its merits. High stands unmodified.

## History
- `#1-connected-true-hides-a-destroyed-edge` `OPEN` reporter — Measured live on a `ProceduralVegetationGraph` in `/Game/PinWrightScratch/PVTest/`: with `DefaultInputNode.In → SeedGenerator_0.In` already wired and confirmed by `pcg.inspect`, a second `pcg.connect_pins {CreatePoints_0.Out → SeedGenerator_0.In}` returned `{"connected":true,"fromNode":"CreatePoints_0","fromPin":"Out","toNode":"SeedGenerator_0","toPin":"In"}` and the following `pcg.inspect` shows the `DefaultInputNode` edge gone — with `edges[]` still length 1, so an edge-count check cannot detect it either. Mechanism traced across three links, all re-derived at HEAD: `UPCGGraph::AddLabeledEdge` (`PCGGraph.cpp:1457-1508`) appends the new edge at `:1488`, calls `ToPin->BreakAllIncompatibleEdges` at `:1495` under `if (!ToPin->AllowsMultipleConnections())` at `:1493`, and **returns exactly that bit** at `:1507`, with the header saying so at `PCGGraph.h:504`; `UPCGGraph::AddEdge` (`:1451-1455`) discards it at `:1453`; and PinWright calls `AddEdge` at `PCGGraphAuthoring.cpp:151` and writes a literal `true` at `:155`, with `:156-159` echoing the request and nothing re-reading the graph. Established that the discarded bool is **unambiguous at this call site** even though it is overloaded in general (`:1462`/`:1470`/`:1478` also return false): the handler proves both pins exist at `PCGGraphAuthoring.cpp:142-149` and both nodes at `:132-138`, so all three failure paths are unreachable and `true` can only mean "an existing edge was destroyed". Established which edge dies and why it is deterministic: `BreakAllIncompatibleEdges` (`PCGPin.cpp:283-369`) walks backwards from `:307` and only sets `bHasAValidEdge` after keeping one (`:364`), and the new edge was appended first, so the survivor is always the new edge and the casualty is always the caller's earlier work; output pins can never be affected (`PCGPin.cpp:444-448` returns true unconditionally for them). Blast radius verified by whole-plugin sweep: `UPVBaseSettings::InputPinProperties` sets `SetAllowMultipleConnections(false)` on the family's default In pin (`PVBaseSettings.cpp:34`), and of 37 `SetAllowMultipleConnections` hits in the PV plugin only two create a `DefaultInputLabel` pin — that one and `PVGrowerSettings.cpp:313`, which bypasses `Super` and re-enforces the same rule — while every `(true)` is on a named auxiliary or output pin; so no PV node's In pin tolerates a second connection, and a 15-40-node plant graph is wired entirely through pins that behave this way. Fix is a one-call swap to `AddLabeledEdge` plus one `SetBoolField("replacedExistingEdge")`, with an optional `replacedEdges[]` from the pin's `Edges` snapshot the handler could take at the same site; `AddLabeledEdge` is C++-only (`PCGGraph.h:505`, no `UFUNCTION`) but `PinWrightPCG` links PCG natively so that costs nothing. Marked **source-only and flagged as the one open question**: whether the displaced edge survives undo — the engine calls `Modify()` on both pins (`PCGPin.cpp:342`, `:352`) so it is undo-ready, but `PCGGraphAuthoring.cpp` opens no `FScopedTransaction` (zero occurrences; one in the whole `PinWrightPCG` module, `PCGGenerateHandler.cpp:156`), so there should be no transaction to record into; that measurement is what would move this to Critical and it was not taken. Recorded but NOT filed here, per the `FoliageHandler.cpp` precedent on `B-foliage-paint-does-no-ground-projection`: the same zero-`FScopedTransaction` finding is namespace-wide across `add_node` / `connect_pins` / `remove_node` and deserves its own ticket. Dedup: searched the board for `connect_pins`, `AddLabeledEdge`, replaced/destroyed edges and single-connection pins — `B-pcg-inspect-edge-direction-reversed` is the read path, `E-pcg-add-node-echo-pin-labels` is response shape on the node-create verbs, `F-pcg-core-graph` `#3` was a since-fixed resolver bug in this same handler, `F-pcg-authoring-parity` is graph parameters, and the `niagara.*` `connect_pins` tickets share no code; nothing on the board mentions a replaced or destroyed edge in any namespace. Cross-linked into the session's recurring class rather than restating it, and to `F-pcg-disconnect-pins` (what makes the loss expensive to repair) and `B-pcg-generate-blind-to-non-point-data` (why nothing downstream would reveal it). Severity High with the Critical reading argued and declined — the write's *effect* is the engine's documented single-connection semantic, visible in the PCG editor as a wire disappearing, so what is defective is the report, not the write; reach declined in both directions.
- `#2-measure-connected-and-name-the-destroyed-edge` `IN-REVIEW` developer — Fixed in `Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphAuthoring.cpp`. Swapped `UPCGGraph::AddEdge` for `UPCGGraph::AddLabeledEdge` (public on 5.3–5.8, same signature, verified across all six installed engines), but the published bits are MEASURED rather than the engine's return: the handler now snapshots the target input pin's upstream endpoints by VALUE (source node id + source pin label — not `UPCGEdge` pointers, which `BreakAllIncompatibleEdges` destroys) before the write, re-resolves the pin and re-reads its edges after, and diffs the two. `connected` is that read-back, not the old literal `true`; `replacedExistingEdge` is the measured bool; `replacedEdges[]` names every destroyed edge with all four endpoints (`fromNode`/`fromPin`/`toNode`/`toPin`), which is what makes the loss repairable with a second `connect_pins`; `warnings[]` carries one line per destroyed edge naming both the new wire and the one it cost. Disclosure, not refusal, per the ticket's own Fix section — no `overwrite` parameter invented. The four top-level `fromNode`/`fromPin`/`toNode`/`toPin` fields are kept and documented as request echoes, so measured and requested are separable in the payload. The engine's `AddLabeledEdge` return is retained as a cross-check rather than published: it is also true for a type-incompatibility break and false on early-out paths this handler has already excluded, so a mismatch against the read-back emits a disagreement warning naming both values and saying the read-back wins (response-honesty convention, `Handlers/Environment/LightingHandler.cpp`). A `connected == false` read-back also warns. UNDO, the ticket's one open question, resolved in the direction the source-only section predicted and fixed: the write now runs inside `FScopedTransaction(NSLOCTEXT("PinWright", "PcgConnectPins", "PCG Connect Pins"))` with `Graph->Modify()`, so the `Modify()` calls the engine already makes on both pins inside `BreakAllIncompatibleEdges` finally have a transaction to record into. Undo was NOT exercised end-to-end — the automation fixture is an `RF_Transient` graph and transient objects are excluded from transaction recording, so an undo assertion there would prove nothing; verifying it needs a saved asset. The namespace-wide missing-transaction gap on `add_node` / `remove_node` is still unfiled, as this ticket's “Noted here, deliberately NOT filed” section intended. Regression test `PinWright.pcg.connect_pins.ReportsReplacedEdge` in `Source/PinWrightPCG/Private/Tests/PCG/TestPCGGraphHandlers.cpp`: two `UPCGCreatePointsSettings` nodes contend for the implicit output node's single input pin, which the fixture forces single-connection (`SetAllowMultipleConnections(false)`, with the 5.3 public-field spelling behind `UE_VERSION_OLDER_THAN(5,4,0)`) rather than depending on the experimental Procedural Vegetation plugin being enabled; confirmed nothing in `AddLabeledEdge`'s notification path calls `UPCGNode::UpdatePins`, so the flag survives both calls. It asserts the first connect reports `replacedExistingEdge:false` with an empty `replacedEdges`, the second reports `connected:true` + `replacedExistingEdge:true` + exactly one `replacedEdges` entry naming all four endpoints of the displaced wire + a non-empty `warnings`, and then asserts the GRAPH matches the claim (first edge gone, second present) so a handler that fabricated `replacedEdges` could not pass. Fails against the pre-fix handler on the absent fields alone. Test id checked with `Content/Python/check_test_ids.py` (CLEAN, no dot-prefix collision with the existing `PinWright.pcg.connect_pins.ResolvesImplicitOutputNode`); no `ErrorCodes::` reference added to a module that has none. `Docs/wiki-src/pcg.md` gained a Conventions bullet stating the destructive semantic, that every PV node's `In` pin is single-connection, the full response shape, the transaction, and that `pcg.inspect`'s edge COUNT cannot detect the loss.
- `#3-namespace-transaction-gap-now-filed` `IN-REVIEW` reporter — **Status deliberately NOT changed; no code touched.** The namespace-wide transaction gap this ticket identified and declined to file ("`PCGGraphAuthoring.cpp` wraps **no** mutation in an `FScopedTransaction` — not `add_node`, not [...] That is a namespace-wide gap deserving its own ticket") is now **`B-pcg-graph-mutators-no-transaction`** (OPEN, Medium), filed under a standing instruction to file everything found. Censused against the current working tree, which carries `#2`'s uncommitted fix: `grep -rn FScopedTransaction Source/PinWrightPCG/` now returns **three** executable occurrences — `PCGGenerateHandler.cpp:156` (`pcg.generate`), `PCGSetNodeProperty.cpp:277` (`pcg.set_node_property`) and `PCGGraphAuthoring.cpp:202` (this ticket's fix) — against **eight** mutating verbs with none: `add_node`, `remove_node`, `add_noise_filter`, `add_slope_filter`, `add_subgraph`, `add_graph_parameter`, `remove_graph_parameter`, `set_self_pruning_settings`. Five of those files contain **zero** `Modify()` calls of any kind, so they would record nothing even inside a transaction. This ticket's own "the displaced edge is probably not recoverable by undo" section is unaffected by the carve-out — its undo question was answered for `connect_pins` by `#2`; the new ticket owns the other eight. Its severity is Medium, anchored to `B-foliage-mutators-no-transaction` (the same property in a different handler family, at the same rating), and does not inherit this ticket's High, which rests on the silent edge destruction rather than on the missing transaction.
