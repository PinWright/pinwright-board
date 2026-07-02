---
id: E-pcg-add-node-echo-pin-labels
title: "pcg.add_node / typed PCG node helpers return only {nodeId} and omit the node's input/output pin labels — forces a separate pcg.inspect round-trip before every connect_pins"
status: OPEN
severity: Low
category: ergonomic
tags: [pcg, add-node, connect-pins, pin-labels, inspect, readback, round-trip, response-shape, consistency]
encounters: 5
lastSeen: 2026-07-02T16:29:33+03:00
---

# PCG node-creating RPCs don't echo the new node's pin labels, so wiring a freshly-added node always costs a separate `pcg.inspect` to discover its pins

Every `pcg.*` handler that creates a node returns only the assigned
`nodeId` (plus a class/kind echo) and **omits the node's input/output pin
labels** — the exact strings `pcg.connect_pins` requires for its
`fromPin`/`toPin` arguments. The pin labels a caller needs are often
**not guessable** from the class name: a surface sampler exposes a
`Surface` input pin (not the default `In`), the graph endpoints use
`In`/`Out`, transform/noise/self-pruning nodes use `In`/`Out`, etc. So
after each `add_node` the caller must fire a separate `pcg.inspect`
just to read back the labels before it can wire — a forced read-back
round-trip baked into the normal author-then-connect flow.

Response shapes today (all under `Handlers/PCG/`):
- `pcg.add_node` → `{nodeId, nodeClass}` (`PCGGraphAuthoring.cpp:85-89`)
- `pcg.add_noise_filter` → `{nodeId, kind}` (`PCGAddNoiseFilter.cpp:211-213`)
- `pcg.add_slope_filter` → `{nodeId}` (`PCGAddSlopeFilter.cpp:135-136`)
- `pcg.add_subgraph` → `{nodeId, subgraphPath}` (`PCGAddSubgraph.cpp:77-79`)

None carries pins. Yet the data is **trivially available at creation
time** — `pcg.inspect` already derives exactly these labels off the
same node via `AppendPinLabels(Node->GetInputPins())` /
`Node->GetOutputPins()` (`PCGGraphInspect.cpp:30-59`). The node object
the create-handler already holds (`Node->GetInputPins()` /
`GetOutputPins()`) is one lambda away from echoing
`inputPins`/`outputPins` in the create response.

## What it should do

`pcg.add_node` and the typed node helpers should echo
`inputPins`/`outputPins` (the same string-array shape `pcg.inspect`
emits per node) in their success response, so the immediately-following
`pcg.connect_pins` can be issued with no intervening read. Cheap and
self-consistent: reuse the `AppendPinLabels` lambda already in
`PCGGraphInspect.cpp` (lift it to `PCGHandlerHelpers.h` so both
surfaces share one implementation). This is an intra-namespace
response-shape consistency gap, not a missing capability — `pcg.inspect`
can already produce the value.

**Workaround:** call `pcg.inspect {graphPath}` once mid-build (or after
each add) to read the actual node ids and pin labels, then wire — which
is exactly what this task did and what the task story had to pre-instruct
("inspect the graph first to read the actual node ids and pin labels,
then wire accordingly").

## Friction evidence (this task — `pcg.create_graph` "RockScatter" build, 13 calls, outcome clean)

The story (step 7's guard clause) explicitly told the agent: "If a pin
label or node id you guess doesn't match, inspect the graph first to
read the actual node ids and pin labels, then wire accordingly." The
agent did exactly one mid-build `pcg.inspect` ("read node ids/pins
before wiring") and, with the labels in hand, all five `pcg.connect_pins`
matched on the first try — `friction:"none — read pin labels via one
mid-build inspect (per task guidance) so every connect_pins matched on
the first try; no retries, no errors, no python fallback."

That "no retries" outcome is *because the task author pre-baked the
inspect*. The friction is the **mandatory extra inspect round-trip**: the
build chain was
`create_graph → add_node ×3 → add_noise_filter → [pcg.inspect to learn pins] → connect_pins ×5 → pcg.inspect`.
The first inspect is pure PROCESS overhead — it exists only to recover
pin labels the four preceding node-creators already knew and discarded.
Without that pre-instructed read, blind wiring would have risked
`connect_pins` failures on the non-default labels (`Surface` on the
surface sampler, the `In`/`Out` endpoints), turning a clean run into a
guess-then-inspect-then-retry loop. The seed `pcg.create_graph` landed
`clean` in the ledger; this is the same "the mutator could have carried
the answer" shape as `E-geometry-deformer-echo-mesh-counts`, applied to
pin labels instead of vertex/triangle counts, and it recurs once per
authored graph (worse the more nodes with non-default pins you chain).

Distinct from the judge's `B-pcg-inspect-edge-direction-reversed` (that
ticket is about `pcg.inspect` reporting *edge direction* reversed; this
is about the *node-creating mutators* omitting pin labels and forcing the
inspect call in the first place). Also distinct from the readback tickets
`E-get-ai-info-no-perception-readback` / `E-game-framework-info-not-asset-readback`
(thin `get_*_info` read verbs that can't surface writes) — here the read
verb (`pcg.inspect`) is fine; the gap is that the create verbs don't
carry the trivially-available pins, so a working read verb gets spammed
once per graph build.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `pcg.create_graph` "RockScatter" rock-scatter graph task (13 calls, outcome clean/tool_bug, friction:"none"; judge filed `B-pcg-inspect-edge-direction-reversed` for the separate edge-direction bug). PROCESS finding: every PCG node-creating RPC returns only `{nodeId}` (+ class/kind echo) and omits the node's pin labels — `pcg.add_node` `{nodeId,nodeClass}` (PCGGraphAuthoring.cpp:85-89), `pcg.add_noise_filter` `{nodeId,kind}` (PCGAddNoiseFilter.cpp:211-213), `pcg.add_slope_filter` `{nodeId}` (PCGAddSlopeFilter.cpp:135-136), `pcg.add_subgraph` `{nodeId,subgraphPath}` (PCGAddSubgraph.cpp:77-79). The labels `connect_pins` needs are non-guessable (surface sampler = `Surface`, endpoints = `In`/`Out`) yet trivially available — `pcg.inspect` already derives them via `AppendPinLabels(Node->GetInputPins()/GetOutputPins())` (PCGGraphInspect.cpp:30-59). Result: the build forced one mid-build `pcg.inspect` purely to recover pin labels before the five `connect_pins`, which the task story had to pre-instruct to keep the run retry-free. Fix: echo `inputPins`/`outputPins` from the create handlers (lift the inspect `AppendPinLabels` lambda into PCGHandlerHelpers.h). Same shape as `E-geometry-deformer-echo-mesh-counts`. Deduped: no existing `E-pcg-*` ticket; the other PCG board files are `B-pcg-inspect-edge-direction-reversed` (edge direction, judge's), `F-pcg-core-graph`/`F-pcg-decompile-ir`/`F-pcg-filters-and-subgraphs` (DONE features) — none addresses create-response pin echo.
- `#2-recurrence-slope-aware-scatter` `OPEN` reporter — Recurrence (2nd task): `pcg.add_slope_filter` "SlopeAwareScatter" foliage-thinning build (11 calls, outcome clean, friction:"none"). Same pattern, same forced read-back: `create_graph → inspect(fresh) → add_node(SurfaceSampler) → add_slope_filter(NormalToDensity_0) → pcg.inspect("read pin labels of new nodes") → connect_pins ×3 → add_slope_filter(NormalToDensity_1) → pcg.inspect`. The 2nd `pcg.inspect` is pure PROCESS overhead — it exists solely to recover the pin labels the just-created `add_node`/`add_slope_filter` already held and discarded, before the three `connect_pins` (Input.In→SurfaceSampler_0.Surface, SurfaceSampler_0.Out→NormalToDensity_0.In, NormalToDensity_0.Out→Output.Out). Confirms the non-guessable label thesis again: the surface sampler's input is `Surface` (not `In`) and the slope filter uses `In`/`Out` — labels the wiring needed but the create responses omitted. The clean/no-retries outcome is again only because the task story pre-instructed "Use inspect first to read the exact node ids and pin labels you need for the connections" (steps 2 and 5); without it, blind wiring would risk a guess-then-retry loop on the `Surface` label. Same fix applies (`pcg.add_slope_filter` → `PCGAddSlopeFilter.cpp:135-136` is one of the named omitting handlers). Recurs once per authored graph; severity stays Low (cosmetic round-trip, never blocks).
- `#3-recurrence-hillside-foliage-scatter` `OPEN` reporter — Recurrence (3rd task): `PG_HillsideFoliageScatter` hillside-foliage build (namespace `pcg`, 26 calls, outcome clean). Same forced read-back, now spanning the full four-node chain: `create_graph → pcg.inspect("initial inspect to learn endpoint pin labels") → add_node(SurfaceSampler_0) → add_slope_filter(NormalToDensity_0) → add_noise_filter(Spatial Noise_0) → add_node(SelfPruning_0) → set_self_pruning_settings → pcg.inspect("read pin labels of all nodes before wiring") → connect_pins ×5 → save → inspect → property.get`. TWO inspects spent purely on pin discovery here (one up-front for the endpoints, one mid-build after all four nodes exist) — both exist only to recover labels the create handlers already held: the surface sampler's `Surface` input (non-default), the slope/noise/self-pruning nodes' `In`/`Out`, and the implicit `DefaultInputNode:In` / `DefaultOutputNode:Out` endpoints. The friction note even frames it as expected ("node pin labels for connect_pins weren't in the wiki, so I inspected the graph after node creation to read the real labels … before wiring — normal read-only discovery"). All five `connect_pins` matched first try BECAUSE the inspects pre-fed the labels — confirming the pattern for the 3rd straight PCG graph build. Same fix (echo `inputPins`/`outputPins` from `pcg.add_node`/`add_slope_filter`/`add_noise_filter`/the SelfPruning `add_node`). Severity stays Low; recurs once-or-twice per authored graph.
- `#4-liveness-rockscatter-decompile-task` `OPEN` reporter — Liveness (4th task, `pcg.decompile` `PCG_RockScatter` build, 17 RPCs): still observed — `create_graph → add_node ×3 → add_slope_filter → pcg.inspect("get ids+pins pre-wire") → connect_pins ×4` again spent one pre-wire `pcg.inspect` purely to recover the node ids and the non-guessable `Surface` input label (the other create responses returned only `{nodeId}`) before all four `connect_pins` matched first try. Pure reconfirm of the same forced-read-back, no new angle; same fix.
- `#5-additional-source-dive-wrong-prediction` `OPEN` reporter — Recurrence + NEW angle (5th task, `HillsideRockScatter` build, namespace `pcg`, 10 real RPCs, outcome clean): `create_graph → add_node(SurfaceSampler_0) → add_slope_filter(NormalToDensity_0) → add_noise_filter(Spatial Noise_0) → pcg.inspect("pre-wire, get endpoint ids+pins") → connect_pins ×4 → asset.save → pcg.inspect`. Same forced pre-wire `pcg.inspect` to recover the labels the four create responses omitted (surface sampler input `Surface`; slope/noise `In`/`Out`; endpoints `DefaultInputNode:In` / `DefaultOutputNode:Out`) before all four `connect_pins` matched first try. NEW EVIDENCE for the non-guessable / echo-the-labels thesis: even though the agent recognized inspect was the authoritative path ("the safest approach is to create the graph and nodes, then run pcg.inspect to see the actual node ids and pin labels before wiring"), it ADDITIONALLY spent ~6 read-only Bash grep/find reads of the UE PCG C++ (PCGSurfaceSampler.cpp/.h, PCGNormalToDensity.h, PCGSpatialNoise.cpp, PCGInputOutputSettings.cpp/.h) trying to pre-predict pin labels — and one prediction was WRONG: from `PCGInputOutputSettings.h` `DefaultInputLabel = TEXT("Input")` the agent concluded the input endpoint's usable output pin is `Input`, but `pcg.inspect` returned `DefaultInputNode` outputPins `['In']`; only the inspect RPC corrected it. This is direct proof that source-diving is an unreliable substitute for the missing echo (it produced a confidently-wrong label), reinforcing that the create handlers should carry `inputPins`/`outputPins` so neither the extra inspect nor the source-spelunking is needed. Same fix (echo pin labels from `pcg.add_node`/`add_slope_filter`/`add_noise_filter` per PCGGraphInspect.cpp `AppendPinLabels`); severity stays Low (round-trip + optional source-dive, never blocks). CallAnalyzer proposed this narrowly as a docs nudge (point the `docs/wiki-src/pcg.md` overlay at `pcg.inspect` for discovering ids+labels, naming the non-default `Surface`/`In`/`Out` labels) — that docs improvement is a valid interim mitigation subsumed by this ticket's echo fix; folding it in here rather than filing a near-duplicate.
