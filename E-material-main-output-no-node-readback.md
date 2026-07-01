---
id: E-material-main-output-no-node-readback
title: "Main material output node is not inspectable: get_material_node_details / material.graph.get_node_details reject the documented 'Main' token their connect_nodes/break_connections siblings accept, and get_material_info omits shadingModel + main-node inputs"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [material, material-authoring, readback, main-node, get-node-details, docs]
---

# The documented "Main" token is write-only: the read RPCs reject it, and get_material_info omits shadingModel + main-node inputs

After authoring a material and connecting nodes into the main material output
(`BaseColor`, `EmissiveColor`, ...), there is no node-detail route to read back
**that the output pins are wired** short of decompiling the whole graph — and the
`"Main"` sentinel the *write* RPCs document and accept is rejected by the *read*
RPCs, an internal contract inconsistency:

- `material.authoring.connect_nodes` / `break_connections` (and the
  `material.graph.*` equivalents) document and accept `targetNodeId`/`nodeId`
  `"(empty or 'Main' for main material node)"` and wire/unwire the main-output
  `FExpressionInput` fields via `MainInputBindings.h`.
- But `material.authoring.get_material_node_details(nodeId:"Main")` →
  `[NOT_FOUND] Node not found.` and
  `material.graph.get_node_details(nodeId:"Main")` →
  `[NODE_NOT_FOUND] Node 'Main' not found. Material has N nodes.` Both read RPCs
  route through `FindExpressionByIdOrName`, which only resolves real
  expressions — the main output node is not a `UMaterialExpression`, so the
  documented `"Main"` token can never resolve on the read side.
- `get_material_info` omits the shading model and the main-node inputs entirely,
  so it cannot answer "what feeds BaseColor / Roughness?" or "what's the shading
  model?" either — even though `create_material` / `set_shading_model` accept a
  `shadingModel` and `twoSided` (which *is* read back), a create/get asymmetry.

The caller is left with only `material.decompile_mgir` (MGIR text IR) to confirm
the output wiring — a heavy whole-graph decompile, not a node-detail call.

> **Scope note.** The separate symptom — `property.get` on a material's
> `BaseColor`/`EmissiveColor` returning `Expression:null` after a successful
> connect — is a `property.get` deprecated-field-shadow issue and is owned by
> OPEN ticket `E-property-get-deprecated-field-silent-null`, which explicitly
> cross-references this one. This ticket is scoped to **Main-node
> inspectability + the get_material_info omissions**; the docs note here points
> agents away from `property.get` and to the now-inspectable `"Main"` node.

## Why it's process friction (clean outcome, but a long detour)

The task completed cleanly, but the "confirm BaseColor is wired" success check
cost a string of dead-end calls before the MGIR fallback worked: a
`get_material_node_details("Main")` → NOT_FOUND, several dead-end reads, a
redundant `connect_nodes` re-connect to retest, then finally
`material.decompile_mgir` as the authoritative readback.

This is the "author then verify" round-trip again, but for the **output stage**
specifically. The DONE ticket `B-material-get-node-details-missing-pins-props`
made non-Main expression nodes inspectable; the Main material node was never
covered, so the last hop of the wiring (node → BaseColor/EmissiveColor) still
had no node-detail readback. The DONE `B-material-main-output-pins-incomplete`
fixed *writing* to extra main inputs (shipping `MainInputBindings.h`); this is
the *reading-back* gap, and it reuses that same table for the read side.

## What it should do

- **Make the main node inspectable.** Accept the documented `"Main"` (or empty)
  sentinel in `get_material_node_details` and `material.graph.get_node_details`
  and emit the main-output inputs as `{name, connectedNodeId, outputIndex, …}`
  by walking `UMaterialEditorOnlyData`'s `FExpressionInput` fields (the same
  `MainInputBindings.h` table `B-material-main-output-pins-incomplete` added for
  the write side drives the read side too). Mirror `BuildExpressionDetailsJson`
  so the payload matches other nodes. This makes the read/write halves agree on
  the node's addressing in **both** namespaces.
- **Add `shadingModel` + the main-output inputs to `get_material_info`** so the
  existing read-back surface answers "what feeds BaseColor / Roughness, and
  what's the shading model?" and the create/get param surfaces are symmetric.
- **Docs:** in `Docs/wiki-src/material.authoring.md` (the `get_material_info`
  workflow note and a Limitations bullet) state that the main node is inspectable
  via the `"Main"` sentinel, that `get_material_info` now reports `shadingModel`
  + `mainInputs[]`, and that `property.get` on a material's
  `BaseColor`/`EmissiveColor` reports `Expression:null` and is **not** a valid
  output-wiring check (use the `"Main"` node or `material.decompile_mgir`).

Filed E-/`ergonomic` (the material is correctly wired and compiles; the gap is
purely in the verification/readback ergonomics + the write/read `"Main"`-token
asymmetry).

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a clean material.authoring.add_switch task (focus material.authoring.add_switch, outcome clean) authoring M_SignBoard_DayNight. The story's success check wanted BaseColor/EmissiveColor confirmed as wired, but get_material_node_details(nodeId:"Main") returns [NOT_FOUND], property.get on UMaterial/EditorOnlyData BaseColor+EmissiveColor returns Expression:null even right after a successful connect_nodes + clean compile_material (and after a re-connect retest), and get_material_info omits shadingModel + main-node inputs. Call cost: 1 Main NOT_FOUND + 5 property.get (4 returning null) + a redundant re-connect + another null read, before material.decompile_mgir gave the authoritative answer and property.get ShadingModel covered the shading model. Distinct from DONE B-material-get-node-details-missing-pins-props (non-Main node payload) and DONE B-material-main-output-pins-incomplete (write side). Propose: make the Main node inspectable (reuse MainInputBindings.h), or add main-output connections + shadingModel to get_material_info; docs minimum — note that property.get Expression:null is not a valid output-wiring check and MGIR is the supported readback. Medium severity — recoverable via MGIR, but the property.get null read is actively misleading (looks unwired when wired).
- `#2-additional-main-alias-doc-inconsistency` `OPEN` reporter — Additional evidence (focus material.authoring.set_blend_mode, outcome clean): independently reproduced on a frosted-glass M_FrostedGlass authoring task. Replay-confirmed live the exact `"Main"` addressing inconsistency, which has a sharper angle than first reported — it is now a **documented-token** inconsistency, not just a missing readback: `connect_nodes`'s own wiki (material.authoring.connect_nodes.md) states `targetNodeId` "(empty or 'Main' for main material node)", and live `connect_nodes(assetPath:/Game/ArchViz/Materials/M_FrostedGlass, sourceNodeId:91F858…, targetNodeId:"Main", inputName:"BaseColor")` → `{"message":"Connected to main material node."}` (accepts the documented 'Main' token), but `get_material_node_details(assetPath:/Game/ArchViz/Materials/M_FrostedGlass, nodeId:"Main")` → `[NOT_FOUND] Node not found.` — the same 'Main' token the sibling write RPC documents and accepts is rejected by the read RPC. Also re-confirmed `get_material_info` omits shadingModel + main-node inputs (live readback returned only domain/blendMode/twoSided/nodeCount/parameters; shadingModel had to come from asset.get_material_stats). Reinforces the proposed fix: when get_material_node_details makes the Main node inspectable, accept the same 'Main' sentinel connect_nodes already documents, so the two halves of the round-trip agree on the node's name.
- `#3-additional-shadingmodel-write-read-asymmetry` `OPEN` reporter — Additional evidence (focus material.authoring.add_noise, outcome clean): third independent repro, in a from-scratch M_ProceduralConcrete authoring task (Surface/Opaque/DefaultLit). Sharpens the **write/read asymmetry** specifically on shadingModel: `material.authoring.create_material` explicitly accepts a `shadingModel` param (wiki lists `Unlit|DefaultLit|Subsurface|...`) and `create_material(name:M_OracleReplayConcrete, path:/Game/Materials, materialDomain:Surface, blendMode:Opaque, shadingModel:DefaultLit)` succeeded — but the matching readback `get_material_info(/Game/Materials/M_OracleReplayConcrete)` returned verbatim `{"domain":"Surface","blendMode":"Opaque","twoSided":false,"nodeCount":0,"parameters":[]}` — i.e. it mirrors the `twoSided` create-param but silently omits the `shadingModel` create-param. So shadingModel is settable at create AND has a `set_shading_model` setter, yet has no readback field anywhere on the get_material_info surface; the agent could only confirm DefaultLit indirectly (set-at-create + clean compile). Reinforces fix bullet #2 (add `shadingModel` to `get_material_info`) — the create/get param surfaces should be symmetric. Separately confirmed NOT a bug: wiring a float2 TextureCoordinate into Noise's float3 World-Position input is accepted by connect_nodes (ok) and only errors at compile_material (`no matching function for call to 'MaterialExpressionNoise'`) — that mirrors UE's own deferred-type-check material editor behavior and compile_material reports it correctly, so no separate ticket.
- `#4-additional-main-rejected-by-graph-get-node-details` `OPEN` reporter — Additional evidence (seed material.graph, outcome clean): fourth independent repro of the `'Main'`-token write/read asymmetry, this time on the `material.graph` namespace specifically. In a hand-authored M_StylizedOcean ocean-surface graph the agent wired `Lerp -> Main.BaseColor` and `Constant -> Main.Roughness` via `material.graph.connect_nodes` (both accepted the `Main` target), but the readback `material.graph.get_node_details {nodeId:"Main"}` was rejected with `[NODE_NOT_FOUND] Node 'Main' not found. Material has 7 nodes.` — same asymmetry as `#2`, now confirmed across BOTH the `material.authoring.get_material_node_details` and the `material.graph.get_node_details` read surfaces (both reject the `Main` token their sibling `connect_nodes`/`break_connections` mutators accept). Process cost / workaround: because the Roughness output wire could not be inspected by node detail after removing the scalar Constant, the agent fell back to `material.graph.break_connections {targetNodeId:"Main", inputName:"Roughness"}` and inferred the wire was already gone from the empty `pinsBroken` result, rather than reading it directly. Friction note verbatim: "get_node_details rejects nodeId='Main' even though connect_nodes/break_connections accept it, so I confirmed the broken Roughness wire via break_connections returning no pinsBroken rather than node-detail inspection." Reinforces the fix bullet: when the Main node is made inspectable, accept the same `Main` sentinel on `material.graph.get_node_details` too (not just the `material.authoring` read), so all read/write RPCs in both namespaces agree on the node's addressing.
- `#5-reword-and-fix` `IN-REVIEW` developer — Reworded (dropped the `property.get`/`Expression:null` symptom from the primary defect — it is owned by OPEN `E-property-get-deprecated-field-silent-null`, which cross-references this ticket — and re-scoped title/body to "make the Main output node inspectable via the documented 'Main'/empty sentinel on both read RPCs + add shadingModel/main-inputs to get_material_info"; category ergonomic). Implemented: (1) `MainInputBindings.h` gained `GetShadingModelString(UMaterial*)` (guards `FMaterialShadingModelField::IsValid()` before `GetFirstShadingModel()`), `BuildMainNodeInputsJson(UMaterialEditorOnlyData*)` (walks the existing 22-entry write-side binding table + CustomizedUVs[0..7], emitting the same `{name, connectedNodeId, outputIndex, mask, maskR/G/B/A}` shape `MGIRExpressionUtils::BuildExpressionDetailsJson` uses), `BuildMainNodeDetailsJson(UMaterial*)` → `{nodeId:"Main", nodeType:"MainMaterialOutput", isMainOutput:true, shadingModel, inputs:[…], outputs:[]}`, and `IsMainNodeId(FString)`. (2) `material.authoring.get_material_node_details` (MaterialAuthoringHandler.cpp) now returns `BuildMainNodeDetailsJson` for the `"Main"` sentinel instead of `[NOT_FOUND]`. (3) `material.graph.get_node_details` (MaterialGraphHandler.cpp) now returns it for an explicit non-empty `"Main"` token (omitted nodeId stays list-all) instead of `[NODE_NOT_FOUND]`. (4) `get_material_info` now emits `shadingModel` + a `mainInputs[]` array. Param-doc strings on both read RPCs now mention `'Main'`. Docs: `Docs/wiki-src/material.authoring.md` workflow note + a new Limitations bullet (Main node inspectable via the sentinel; property.get BaseColor is not a valid wiring check). No AssetDumpCache aspect bump — these are live RPC payloads, not a dump aspect. Files: `Source/.../Handlers/Material/MainInputBindings.h`, `Source/.../Handlers/Material/MaterialAuthoringHandler.cpp`, `Source/.../Handlers/Material/MaterialGraphHandler.cpp`, `Docs/wiki-src/material.authoring.md`. Test: new `Source/.../Tests/Material/TestMaterialMainNodeReadback.cpp` with three regressions exercising production handlers — `material.graph.get_node_details{nodeId:"Main"}` and `material.authoring.get_material_node_details{nodeId:"Main"}` each resolve to success (not NODE_NOT_FOUND/NOT_FOUND) with `nodeId:"Main"`/`isMainOutput`/`shadingModel:"DefaultLit"` and the wired Roughness input carrying the param GUID, and `get_material_info` returns `shadingModel:"DefaultLit"` + a `mainInputs[]` Roughness entry; reverting any of the three handler branches fails the matching assertions.
