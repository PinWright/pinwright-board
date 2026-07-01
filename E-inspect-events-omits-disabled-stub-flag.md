---
id: E-inspect-events-omits-disabled-stub-flag
title: "blueprint.inspect events[] omits an enabled flag, hiding disabled default stub events"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [blueprint, inspect, events]
encounters: 3
lastSeen: 2026-06-29T01:08:37Z
---

# blueprint.inspect events[] omits an enabled flag, hiding disabled default stub events

`blueprint.inspect`'s `events[]` array lists the standard Actor default ghost
stubs (`ReceiveActorBeginOverlap`, `ReceiveTick`) **flat and indistinguishable**
from a real authored handler. Each entry is just `{name, eventType}` — there is
no `enabled`/`disabled` (or `bodyless`/`stub`) field — so a caller that trusts
`inspect` to answer "what events does this Blueprint handle?" concludes the BP
handles `Tick` and `ActorBeginOverlap` when those are inert, body-less, disabled
placeholder nodes that do nothing. The same flat `events[]` is shared by
`blueprint.get` (both go through `CollectBlueprintEvents`).

This bites the very common "verify exactly N events after authoring" pattern
(e.g. idempotency / no-duplicate-accumulation checks): a fresh Actor BP created
via `blueprint.create` always carries these two disabled stubs, so every such BP
reports two phantom lifecycle events the author never wrote, which can be
mistaken for accumulated/duplicate entries.

**Scope (narrowed):** this ticket is about `blueprint.inspect` (and the shared
`blueprint.get`) `events[]` **only**. The graph-node layer is already solved:
`blueprint.graph.get_nodes` with `includeNodeState:true` emits a structured
`nodeState` block — `isEnabled` (bool), `enabledState:"disabled"`,
`isDisabledByUser` — via `BuildNodeStateJson`
(`BlueprintGraphInspectionHandler.cpp:79-113`). So the disabled signal is NOT
"comment-only" and get_nodes does NOT lack a first-class boolean (History `#2`'s
premise is outdated — see the correction below). `nodeState` is opt-in
(`includeNodeState` defaults false) and is not a `fields`-projection key, which
is why a projected get_nodes call appeared to drop it; that is projection
ergonomics, not a missing capability. The remaining actionable gap is the
summary readback's `events[]`.

**Fix:** in `CollectBlueprintEvents` (`BlueprintHandlerUtils.cpp`), have the
`AppendEvent` lambda add an `enabled` boolean to each event entry, derived from
the node already in hand — `SourceNode->IsNodeEnabled()` — matching the
`nodeState.isEnabled` signal the graph-inspection family already exposes. The
inspect text formatter (`BlueprintInspectFormatter.cpp`) should surface it as a
`[disabled]` marker alongside the existing `[custom]` marker. Optional docs
companion: note on the `blueprint.inspect` wiki page (and
`blueprint.bpir-gotchas.md`) that a fresh Actor BP's `events[]` includes the
standard disabled default stubs.

**Verbatim repro** (`/Game/BP_IdemReUpsert`, fresh Actor BP with one authored
`BeginPlay` chain + one custom event `OnPing`):

`call("blueprint.inspect", {assetPath:"/Game/BP_IdemReUpsert"})` →
```json
"events":[
  {"name":"ReceiveActorBeginOverlap","eventType":"K2Node_Event"},
  {"name":"ReceiveTick","eventType":"K2Node_Event"},
  {"name":"ReceiveBeginPlay","eventType":"K2Node_Event"},
  {"name":"OnPing","eventType":"custom","parameters":[{"name":"Count","type":"int"}]}
]
```
All three `K2Node_Event` entries look identical, but `call("blueprint.graph.get_nodes",
{assetPath:"/Game/BP_IdemReUpsert", graphName:"EventGraph"})` shows the first two
are inert:
```json
{"nodeTitle":"Event ActorBeginOverlap","comment":"This node is disabled and will not be called.\nDrag off pins to build functionality.", ... "then" ... "linkedTo":[]}
{"nodeTitle":"Event Tick","comment":"This node is disabled and will not be called.\nDrag off pins to build functionality.", ... "then" ... "linkedTo":[]}
```
while `Event BeginPlay` has `comment:""` and its `then` pin links into the
authored body. `inspect` exposes none of this.

Distinct from `E-compile-bpir-idempotent-omits-guid-regen` (OPEN — same ghost
stubs mentioned, but that ticket is about `compile_bpir` docs omitting GUID
regeneration; this is about `inspect`'s events readback dropping the disabled
state). No existing inspect-events ticket covers this.

## History
- `#1-initial-repro` `OPEN` reporter — Found during a `blueprint.compile_bpir` idempotency seed probe (the compile_bpir idempotent re-upsert contract itself held cleanly — byte-identical decompile, stable 12-node count, orphanedCount 0). Replay-confirmed on `/Game/BP_IdemReUpsert`: `blueprint.inspect` lists `ReceiveActorBeginOverlap` and `ReceiveTick` in `events[]` as plain `K2Node_Event` with no disabled marker, while `blueprint.graph.get_nodes` on the same graph shows both carry `"This node is disabled and will not be called."` and empty `then` links. Caller trusting `inspect` cannot tell the inert default stubs from the one real authored `BeginPlay` handler. Culprit method: `blueprint.inspect` (neighbor of the seed `blueprint.compile_bpir`).
- `#2-getnodes-projection-drops-disabled-signal` `OPEN` reporter — Cross-task reach reinforcement of the same phantom-stub "verify exactly N" tax, this time via `blueprint.graph.get_nodes` rather than `inspect`. An authored-position round-trip-equivalence probe (focus `blueprint.compile_bpir`, fresh Actor BP `/Game/BP_BpirPosRoundTrip`) called `get_nodes` with `fields=[nodeId,nodeType,nodeTitle,x,y]` and got **8** nodes — the 6 authored plus the 2 disabled Actor default stubs (`ActorBeginOverlap`, `Tick`) — with **no structured enabled/disabled flag** in the projected output, forcing the agent to manually reconcile the get_nodes count (8) against the decompiled body (6) to confirm no phantom helpers were injected (friction note verbatim: the *"count check needed care to interpret"*). Distinct facet from `#1`: the disabled signal that `#1` credits `get_nodes` with surfacing lives ONLY in the free-text `comment` string, which a structured `fields` projection silently drops — so even `get_nodes` lacks a first-class disabled boolean. Reinforces that the proper fix is a STRUCTURED `enabled`/`isStub` flag across the whole readback family (`inspect` `events[]` AND `get_nodes`), not just the unstructured comment. No call cost this run (the agent interpreted it correctly and the stubs correctly stayed out of the decompiled body); reach +1, severity unchanged. Dedup: matched this OPEN ticket on rg `disabled`/`stub`/`get_nodes count`; appended rather than re-filed.
- `#3-decompile-roundtrip-docs-facet` `OPEN` reporter — Cross-task reach + a **docs** facet on the same fresh-Actor-BP default-stub trap, this time via `blueprint.decompile` (struggle audit, focus `blueprint.compile_bpir`, `/Game/BP_RoundTripProbe`). The trap is undocumented where it bites: `docs/wiki-src/blueprint.bpir-gotchas.md` has NO mention of "stub"/"disabled"/"default event" (grep returned nothing), and `blueprint.decompile.md` does not warn that a freshly-created Actor BP decompiles with 3 default DISABLED event stubs that will appear in any decompile-based round-trip comparison and are NOT removed by `compile_bpir` append. The agent had to *discover* this empirically by decompiling the empty baseline, which then drove a ~6-call per-node delete detour (see sibling `F-graph-batch-delete-clear-mode`). **Docs page(s) to improve:** `docs/wiki-src/blueprint.bpir-gotchas.md` and/or `docs/wiki-src/blueprint.decompile.md` — add the warning plus the clean-slate recipe (delete the 3 stubs, or start blank). Reinforces that this is the canonical first step for the round-trip-fidelity tests this suite runs repeatedly. Reach +1; severity unchanged (the structured-flag fix is the primary remedy, this is the docs companion). Dedup: matched this OPEN ticket on rg `default stub`/`disabled`/`fresh actor`; appended rather than re-filed.
- `#4-reword-and-fix` `IN-REVIEW` developer — Reworded then fixed. **Reword:** narrowed scope to `blueprint.inspect` / `blueprint.get` `events[]` only, and corrected History `#2`'s outdated premise. `get_nodes` already exposes a first-class structured `nodeState.isEnabled` (plus `enabledState:"disabled"`, `isDisabledByUser`) via `includeNodeState:true` — `BuildNodeStateJson` (`BlueprintGraphInspectionHandler.cpp:79-113`); the signal is NOT comment-only, so the get_nodes half of the ask is already shipped (corroborated by all three validity lenses). The reporter saw "no flag" only because `nodeState` is opt-in and not a `fields` key — projection ergonomics, not a code gap. **Fix:** `CollectBlueprintEvents`'s `AppendEvent` lambda now sets `enabled` = `SourceNode->IsNodeEnabled()` on every `events[]` entry (`Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp`), so the inert disabled default stubs read `enabled:false` and live handlers read `enabled:true`; shared by both inspect and get via `BuildBlueprintSnapshot`. The inspect text formatter appends a `[disabled]` marker beside the existing `[custom]` marker (`BlueprintInspectFormatter.cpp`). **Test:** `PinWright.blueprint.inspect.EventsCarryEnabledFlag` (`Source/PinWright/Private/Tests/Blueprint/TestBlueprintHandlers.cpp`) seeds an enabled `ReceiveBeginPlay` and a disabled `ReceiveTick` event, drives `blueprint.inspect`, and asserts each `events[]` entry carries `enabled` with the disabled stub false and the live handler true (reverting the `SetBoolField` fails the presence assertions). Docs companion (gotchas/wiki note) left as an optional follow-up. Not yet compiled/tested — a later phase verifies.
</content>
</invoke>
