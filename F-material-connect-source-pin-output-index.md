---
id: F-material-connect-source-pin-output-index
title: "connect_nodes ignores sourcePin / OutputIndex — multi-output nodes unaddressable"
status: DONE
severity: High
category: feature
tags: [material, material-graph, material-authoring, connect-nodes, multi-output]
---

# Source output pin is silently dropped on connect

`material.authoring.connect_nodes` and `material.graph.connect_nodes` both
take a `sourceNodeId` and connect it to the target's input by setting:

```cpp
Input->Expression = SourceExpr;
// Input->OutputIndex is never set
// Input->Mask / MaskR / MaskG / MaskB / MaskA never touched
```

`material.authoring.connect_nodes` accepts an optional `sourcePin`
parameter (declared as `RPC_PARAM_OPT("sourcePin", "string", "Output pin
name on source node")`) but **never reads it** — `grep`ing the handler
file shows `sourcePin` appears only in the param-spec declaration.
`material.graph.connect_nodes` doesn't even declare the parameter.

`FExpressionInput::OutputIndex` selects which of a source expression's
outputs feeds the wire. For single-output expressions
(`Add`, `Multiply`, `ScalarParameter`) it's always 0 and the omission is
harmless. For multi-output expressions it's load-bearing:

- `BreakOutFloat4` — outputs `R`, `G`, `B`, `A` (indices 0–3)
- `BreakOutFloat3` — outputs `R`, `G`, `B`
- `MakeFloat3` — single output (fine)
- `ComponentMask` — single output (handled via the source's mask, fine)
- `MaterialExpressionMaterialFunctionCall` — one output per
  `FunctionOutputs[]` entry (zero up to N)
- `SceneTexture` — multiple sample outputs
- `MaterialAttributes` getter/setter expressions — one output per
  attribute slot

For all these, the current API silently produces an `OutputIndex = 0`
connection, which means callers cannot, e.g., wire the `G` channel of a
`BreakOutFloat4` to `Roughness`. The only workaround is `ComponentMask`,
which means an extra node and a second connect call.

**Fix:**

1. Add `sourcePin` (string) and `sourceOutputIndex` (int) to **both**
   `material.authoring.connect_nodes` and `material.graph.connect_nodes`.
   Resolve `sourcePin` against `SourceExpr->GetOutputs()` (UE returns
   a `TArray<FExpressionOutput>` with `OutputName` per entry); fall back
   to `sourceOutputIndex` when the name doesn't match. Set
   `Input->OutputIndex = ResolvedIndex` before `Material->PostEditChange()`.
2. Optional: expose mask routing — `sourceMask: "R"|"G"|"B"|"A"|"RGB"|"RGBA"`
   that sets `Input->Mask`, `MaskR/G/B/A` accordingly. This subsumes the
   common `ComponentMask` insertion pattern.
3. Cross-ref `B-material-get-node-details-missing-pins-props` — the
   inspection-side `outputs[]` list is needed for callers to discover
   names by which to wire.

Note: this is a *feature* (the connection API works for single-output
nodes; multi-output is an unimplemented capability) rather than a *bug*,
hence `F-`. But `material.authoring.connect_nodes` declaring a `sourcePin`
param it never reads is borderline — could also be a `B-` if the team
prefers.

## Repro

1. Create material `M`. Add `BreakMaterialAttributes` expression
   (or `MakeFloat4`/`BreakOutFloat4` chain) — node `B`.
2. `material.graph.connect_nodes(materialPath: M, sourceNodeId: B,
   sourcePin: "G", targetNodeId: "", inputName: "Roughness")` — wires
   the `R` (index 0) output, not `G`.
3. Verify in editor: the wire enters the wrong output socket.

## History
- `#1-source-pin-dropped` `OPEN` reporter — Material API audit caught both `material.authoring.connect_nodes` and `material.graph.connect_nodes` setting `Input->Expression` without touching `Input->OutputIndex`. The authoring variant declares a `sourcePin` parameter it never reads; the graph variant lacks the parameter entirely. Result: multi-output nodes (BreakOutFloat4, BreakMaterialAttributes, MaterialFunctionCall with multiple outputs, SceneTexture sample variants) are forced to output index 0. Add `sourcePin` (resolved against `GetOutputs()`) and `sourceOutputIndex` (integer fallback) to both handlers; optionally expose `sourceMask` as a typed alternative to inserting a `ComponentMask` node.
- `#2-reviewed-and-confirmed` `OPEN` reporter — Verified claims against source. `MaterialAuthoringHandler.cpp` line 1255 declares `RPC_PARAM_OPT("sourcePin", ...)` but the handler body (lines 1259–1336) only reads `sourceNodeId`/`targetNodeId`/`inputName`; `sourcePin` is never pulled from `Ctx`. Eleven assignment sites set `*.Expression = SourceExpr` (main-material branch lines 1279–1300, expression branch line 1326) with no `OutputIndex` write. `MaterialGraphHandler.cpp` lines 184–187 confirm `sourcePin` is not in the param spec at all; lines 221–230 and 256 reproduce the same `Expression`-only assignment pattern. No duplicate ticket — sibling material entries cover unrelated issues (`B-material-break-connections-named-pin-noop`, `B-material-get-node-details-missing-pins-props`, `B-material-main-output-pins-incomplete`, etc.). Severity `High` is appropriate: silently routes wrong output for every BreakOutFloat3/4, BreakMaterialAttributes, multi-output MaterialFunctionCall, and SceneTexture sample, with no error signal to the caller. Fix is small but not literally one line: add the missing param to `material.graph.connect_nodes`, read `sourcePin`/`sourceOutputIndex` in both handlers, resolve via `SourceExpr->GetOutputs()`, and assign `OutputIndex` at all eleven inlined assignment sites — best done by extracting a small `ApplyOutputIndex(FExpressionInput&, SourceExpr, sourcePin, sourceOutputIndex)` helper.
- `#3-apply-connection-helper` `IN-REVIEW` developer — Added inline ApplyConnection(FExpressionInput&, UMaterialExpression*, FString SourcePinName, int32 SourceOutputIndex) helper in Handlers/Material/MaterialFinders.h. Resolves SourcePinName against SourceExpr->GetOutputs()[i].OutputName case-insensitive, falls back to SourceOutputIndex when name unmatched or empty. Wired into both material.authoring.connect_nodes (added sourceOutputIndex param) and material.graph.connect_nodes (added both sourcePin and sourceOutputIndex params). sourceMask deferred. Regression test TestMaterialConnectSourcePin.cpp asserts BreakMaterialAttributes 'Roughness' output resolves to the correct non-zero OutputIndex.
- `#4-verify-source-pin-output-index` `DONE` tester — Verified: `material.graph.connect_nodes` and `material.authoring.connect_nodes` wiki schemas expose `sourcePin` and `sourceOutputIndex`; source inspection confirms both handlers read those fields and call `ApplyConnection`, which resolves named outputs/fallback indices and assigns `Input.OutputIndex`; regression test `TestMaterialConnectSourcePin.cpp` asserts BreakMaterialAttributes `Roughness` resolves to a non-zero output index and `sourceOutputIndex=2` sets Metallic.
