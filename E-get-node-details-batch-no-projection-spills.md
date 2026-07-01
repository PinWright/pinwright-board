---
id: E-get-node-details-batch-no-projection-spills
title: "blueprint.graph.get_node_details_batch has no fields projection, so even a 6-node positions cross-check returns the full property set and overflows the inline budget to disk"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, graph, get_node_details_batch, response-size, oversized, projection, response-spill]
encounters: 4
lastSeen: 2026-06-28T19:14:55Z
---

# `blueprint.graph.get_node_details_batch` always returns the full per-node property set, so a tiny batch spills

`blueprint.graph.get_node_details_batch` (`nodeIds: ["<id>", ...]` → details for
every node in one call) always emits the **complete property set per node**.
There is no narrowing lever — no `fields` projection and no positions-only / summary
mode — so the response size scales with (nodes × properties) and the caller cannot
ask for the lightweight shape an intent like "verify the authored coordinates"
actually needs.

The practical effect: a **6-node** batch already crosses the 10000-char inline
budget, returns `outputTooLong`, and the full payload is written to
`Saved/EditorAutomation/HttpResponses/.../<uuid>.json` — forcing the caller to
`Read` the spilled file just to recover two integer fields per node
(`NodePosX`/`NodePosY`). The spill mechanism worked as designed; the friction is
that the per-node detail verb has no way to stay inline, so the spill fires on
batches far smaller than anyone would expect when the caller wanted only a couple
of fields.

## What it should do

Give `get_node_details_batch` a narrowing lever so a small targeted cross-check
stays inline, mirroring the projection lever proposed for its sibling
`get_nodes` (`E-get-nodes-pins-spill-no-projection`):

- Accept an optional `fields` projection (e.g. `fields: ["NodePosX", "NodePosY"]`)
  or a positions-only / summary flag that returns just the requested fields and
  **omits the rest of the property set** unless explicitly requested. A
  coordinate or single-property cross-check of a handful of nodes would then stay
  inline instead of spilling.

The verbose full-fidelity shape stays available opt-in for callers that want the
complete property dump.

## Distinct from

- `E-get-nodes-pins-spill-no-projection` (OPEN) — **same fix family** (add a
  `fields`/`namesOnly` projection so a tiny input stays inline) but a **different
  method**: that ticket is `blueprint.graph.get_nodes` (full `pins`+`linkedTo`
  per node on the enumerate path); this ticket is `get_node_details_batch` (full
  property set per node on the per-node detail/verify path). Both lack a
  projection lever; neither one's fix shrinks the other's payload.
- `E-get-node-details-batch-undiscovered-on-read-path` (WONTFIX) — same method,
  but that is a docs/discoverability gap (agents reach for singular
  `get_node_details` N times instead of the batch); this is the response-size /
  no-projection gap on the batch method itself.
- `E-pin-details-batch-requests-shape-mismatch` (OPEN) — the `get_pin_details_batch`
  vs `get_node_details_batch` param-shape divergence (a misuse-then-correct), not
  a response-size issue.
- `E-http-response-spill` (DONE) — the generic spill-to-disk mechanism; this
  ticket is that the method has no lever to avoid triggering it.

## Evidence

From the struggle audit of a clean-process BPIR positioned-custom-event
round-trip probe (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome
`tool_bug` filed as `B-bpir-fallthrough-reconverge-dropped` — unrelated to this
spill; 14 RPCs, all `ok`/non-error). To cross-check the authored `@(x, y)`
coordinates against the actual `NodePosX`/`NodePosY` for the **6** BPIR-created
nodes on `/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic`,
the agent called `blueprint.graph.get_node_details_batch`, which returned
`{"outputTooLong":true,"message":"Response exceeds display limit (25759 chars,
threshold 10000); full payload written to .../HttpResponses/.../...json",...}`.
The full per-node details for just 6 nodes (**25759 chars = ~2.6× the 10000
budget**) blew the inline threshold and spilled to disk, forcing an extra `Read`
of the `HttpResponses` JSON purely to read back two integer fields per node. The
call-trace analyzer flagged this as the run's secondary inefficiency ("workaround"
on `blueprint.graph.get_node_details_batch"): minor (one extra round-trip) but
recurring for any positional-fidelity / single-property probe.

Severity Low: by the rubric a response-spill that only forces a `Read` is Low;
the per-node detail-verify path is a recurring but not every-session path, so no
reach bump.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean-process BPIR positioned-custom-event round-trip probe (focus `blueprint.compile_bpir`, namespace `blueprint`, 14 calls all `ok`/non-error; the task's own outcome was the unrelated compile bug `B-bpir-fallthrough-reconverge-dropped`). To cross-check authored `@(x,y)` against `NodePosX`/`NodePosY` for the **6** BPIR-created nodes on `BP_Light_Bulb_Basic`, `blueprint.graph.get_node_details_batch` returned `outputTooLong` at **25759 chars** (threshold 10000) and spilled the full payload to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing an extra `Read` to recover just two integer fields per node. Root cause: the method always emits the complete property set per node with no `fields` projection / positions-only lever, so even a 6-node batch overflows when the caller wanted only `NodePosX`/`NodePosY`. Proposed: add an optional `fields` projection (or positions-only/summary flag) so a small targeted cross-check stays inline; verbose full dump stays opt-in. Same fix family as `E-get-nodes-pins-spill-no-projection` but a different method (per-node detail verb vs the enumerate verb). Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — no ticket pairs `get_node_details_batch` with a projection/spill fix; `E-get-node-details-batch-undiscovered-on-read-path` (discoverability, WONTFIX), `E-pin-details-batch-requests-shape-mismatch` (param-shape), and `E-get-nodes-pins-spill-no-projection` (different method) are all distinct.
- `#2-cross-task-evidence` `OPEN` reporter — Recurrence from a SECOND clean-process positioned-BPIR round-trip probe (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome clean — no tool bug filed this iteration). Same friction, same target: to cross-check authored `@(x,y)` against on-disk `NodePosX`/`NodePosY` for the **6** BPIR-created nodes of new event `RTProbe_Positioned_7741` on `BP_Light_Bulb_Basic`, `blueprint.graph.get_node_details_batch({nodeIds:[6 ids]})` returned `{outputTooLong:true, characters:22007, threshold:10000}` and spilled the full payload to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing **two** follow-up Greps over the spilled file to recover x/y per node. Now **two independent probes** hit the same wall (25759 and 22007 chars for a 6-node batch), and the call-trace analyzer flagged it as this run's PRIMARY `workaround` inefficiency — confirming the round-trip-equivalence oracle the task goal explicitly recommends is exactly the positions-only use case that needs the proposed `fields` / positions-only projection. Wiki (`blueprint.graph.get_node_details_batch.md`) still lists only `assetPath`/`nodeIds`/`graphName` — no projection lever. Severity unchanged Low; reach is now stronger (the spill recurs across tasks whenever a positional-fidelity cross-check touches >~5 nodes). Dedup: matched this OPEN ticket on rg; appended rather than re-filed.
- `#3-cross-task-evidence-4-node-batch` `OPEN` reporter — THIRD independent recurrence, and a **smaller-batch** data point: a clean-process BPIR round-trip-equivalence probe (focus `blueprint.compile_bpir`, namespace `blueprint`; fresh Actor BP `/Game/BP_BpirRoundTrip`, 7 RPCs all `ok`/non-error; the iteration's own outcome was the unrelated `B-bpir-fallthrough-reconverge-dropped`). To verify branch/reconvergence positions+pins the agent batched just **4** nodeIds (`2A21B3A9…`, `3FCD8A94…`, `46ED3C14…`, `2E40FADF…`) — branch + Valid + Reconverged + Invalid — and `blueprint.graph.get_node_details_batch` returned `{outputTooLong:true, characters:18823, threshold:10000}`, spilling the full payload to `Saved/EditorAutomation/HttpResponses/.../20260628T182025Z_34e5152d-….json` and forcing an out-of-band `Read` of the spill file to recover the data it asked for. **18823 chars for only 4 nodes (~1.9× the 10000 budget)** lowers the known overflow floor below the prior 6-node evidence (25759 / 22007): even a minimal 4-node wiring/position cross-check — exactly the round-trip oracle the task goal recommends — cannot stay inline without the proposed `fields` / positions-only projection. Severity unchanged Low (spill-only). Dedup: matched this OPEN ticket on rg `get_node_details_batch`/`outputTooLong`; appended rather than re-filed.
- `#4-cross-task-evidence-2-node-floor` `OPEN` reporter — FOURTH independent recurrence and a NEW low-water mark: even a **2-node** batch overflows. From the struggle audit of a clean-process BPIR positioned fork/reconverge round-trip probe (focus `blueprint.compile_bpir`, namespace `blueprint`; 13 RPCs all `ok`/non-error; the iteration's own outcome was the unrelated `B-bpir-fallthrough-reconverge-dropped`). To confirm two nodes' titles+positions (`584F78DD…`='false arm'@(759,419), `94ED5849…`='reconverged'@(1087,253)) the agent batched just **2** nodeIds and `blueprint.graph.get_node_details_batch` returned `{outputTooLong:true, characters:11068, threshold:10000}`, spilling to `Saved/PinWright/HttpResponses/…json` and forcing a python parse of the spill to pull only `structuredContent.items[].details {title,x,y,InString}`. The bloat is the full per-pin default set repeated per node (`bPrintToScreen`, `bPrintToLog`, `TextColor: FLinearColor(0.0,0.66,1.0,1.0)`, `Duration: 2.0`, …). **11068 chars for only 2 PrintString nodes (~1.1× the 10000 budget)** drops the known overflow floor below the prior 4-node datapoint (18823 chars, #3): the projection lever is needed even for the smallest possible position/wiring cross-check. Severity unchanged Low (spill-only). Dedup: matched this OPEN ticket on rg `get_node_details_batch`/`outputTooLong`; appended rather than re-filed.
