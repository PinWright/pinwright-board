---
id: F-metasound-decompile-msir
title: "Add MetaSound decompile-only text IR (MSIR)"
status: DONE
severity: Medium
category: feature
tags: [metasound, ir, decompile, audio, audio-authoring]
---

# Add MetaSound decompile-only text IR (MSIR)

The plugin exposes 22+ imperative MetaSound authoring RPCs under
`audio.authoring.*` (create, graph mutate, I/O, variables, interfaces,
presets) but **zero decompile path**. Agents inspecting an existing
MetaSound asset must either crawl the JSON dump or piece state together
through repeated `audio.search.*` / `audio.authoring.describe_metasound`
calls — neither produces a single readable artifact suitable for diffing,
review, or wiki documentation. The current `audio.authoring.md` wiki page
explicitly states "MetaSound has no text IR"; that claim must be revised
when this ticket lands.

Add **MSIR**: a decompile-only flat node + wire text IR analogous to
AGIR/MGIR, emitted both as an RPC result and as a per-asset sidecar
(`msir.txt`) by `asset.dump`. **Decompile only** — no authoring/compile
back-channel in the initial ticket; round-trip authoring is a future
follow-up once the surface stabilises.

## Grammar (flat node + wire form)

```
metasound MS_Engine kind=Source {
  interface `MetasoundV1.0.SourceInterface` v=1.0
  interface `MonoSource.AudioOutput` v=1.0

  input float    RPM        = 1500.0
  input wave     Wave       = "/Game/Audio/SFX_Engine"
  output audio   OutAudio
  output trigger OnFinished

  variable float CurrentGain = 1.0  // id=v1

  node n1 = `WaveTable.Oscillator` v=1.0 @(120, 80)
  node n2 = `Math.Multiply.Audio`   v=1.0 @(320, 80) { A_default: 0.5 }
  node n3 = `Variables.SetFloat`   v=1.0 @(320, 200) { variable: $v1 }

  wire $RPM       -> n1.Frequency
  wire $Wave      -> n1.WaveTable
  wire n1.Audio   -> n2.A
  wire n2.Result  -> $OutAudio
}

metasound MSPreset_Boss kind=SourcePreset based_on="/Game/Audio/MS_Engine" {
  override input float Volume = 0.7
}
```

Design choices:
- Class reference: backtick-quoted `Namespace.Name v=Major.Minor` — resolved
  per-document via `FMetasoundFrontendDocument::Dependencies[]` registry.
- Sigils: `$Name` for graph I/O and variable refs; `nN` for decompiler-assigned
  node locals.
- `kind=` discriminates `Source` / `Patch` / `SourcePreset` / `PatchPreset`,
  driven from `RootGraph.PresetOptions.bIsPreset` + class metadata.
- Per-node vertex literals (`InputLiterals[]` keyed by VertexID) collapse to
  per-input-name fields in emitted text.
- Node positions read from
  `FMetasoundFrontendNodeStyle.Display.Locations[DefaultPageID]`.

## Engine traversal

Canonical path: `IMetaSoundDocumentInterface::GetConstDocument()` →
`FMetasoundFrontendDocument` → `RootGraph` (`FMetasoundFrontendGraphClass`) →
`DefaultInterface` (Inputs/Outputs) → page-scoped `Graph` (Nodes/Edges/
Variables); per-node `FMetasoundFrontendNode` with `ClassID` resolving
through `Doc.Dependencies[]`.

`MetaSoundDumpBuilder.{h,cpp}` (220 LoC under `Private/Handlers/Asset/`)
already walks this exact structure for the JSON dump — covers ~60% of the
decompiler work. Reuse its node/edge/variable/dependency walks; the new
MSIR layer only adds text emission.

## Implementation sketch

New files:
- `Private/MSIR/MSIRTypes.h` — `FMSIRDocument` / `FMSIRNode` / `FMSIRWire` /
  `FMSIRInput` / `FMSIROutput` / `FMSIRVariable` / `FMSIRClassRef` POCOs.
- `Private/MSIR/MSIRDecompiler.{h,cpp}` — frontend doc → MSIR POCO →
  text emitter.
- `Private/Handlers/Audio/MetaSound/MetaSoundDecompileHandler.cpp` — registers
  `audio.authoring.decompile_metasound` (input: asset path; output: msir
  text + structured POCO mirror).

Reuse:
- `MetaSoundDumpBuilder.cpp` document/dependency/node/edge traversal.
- `IrCore/IrTextUtils` (`Quote`, `FormatNameToken`, `FormatPositionSuffix`,
  `FormatFieldList`) — same emit primitives used by AGIR/MGIR.
- AGIR's decompile-only RPC + sidecar registration pattern as a shape
  template.

Sidecar: add `msir.txt` to `AssetDumpHandler.h` `DumpFileNames`, slotted
between the existing MetaSound JSON dump and AGIR.

Effort: ~400 LoC decompiler / ~900 LoC total including tests + wiki,
estimated 3–5 dev-days. Smaller than MGIR's decompile-only path because
the MetaSound frontend document is already well-typed: no expression
trees, no per-node-class custom logic — every node resolves through the
class registry uniformly.

## Wiki impact

`docs/wiki/audio.authoring.md` currently asserts "MetaSound has no text IR".
That statement is invalidated by this ticket and **must be replaced** in the
same change as part of the IN-REVIEW handoff: add an MSIR section mirroring
the AGIR/MGIR write-ups (grammar pointer, decompile-only scope, sidecar
filename, RPC name, round-trip-authoring marked as future work).

## Soft prerequisites

- `F-ircore-reflected-property-emit` — shared reflected-property emit path
  used by all IR emitters; landing first reduces MSIR-specific literal
  formatting code.
- `R-ir-grammar-harmonize` — keeps MSIR's class-ref / sigil / position-suffix
  syntax aligned with AGIR/MGIR/BPIR conventions.
- `F-ir-authoring-guide` — once authoring follow-up exists, the guide gains
  an MSIR chapter; not blocking for the decompile-only initial drop.

None are hard blockers — MSIR can ship standalone and pick up the shared
infra retroactively.

**Fix:** implement decompiler + RPC + sidecar + wiki update as outlined
above. Defer round-trip authoring (`audio.authoring.compile_msir`) to a
follow-up ticket gated on real agent demand.

## History
- `#1-initial-proposal` `OPEN` reporter — Filed: zero decompile path exists for MetaSound today (only 22+ imperative authoring RPCs); proposing decompile-only MSIR text IR + `audio.authoring.decompile_metasound` RPC + `msir.txt` sidecar. Reuses `MetaSoundDumpBuilder` traversal (~60% of work) and `IrCore/IrTextUtils` emit primitives. Wiki claim "MetaSound has no text IR" in `audio.authoring.md` must be updated when this lands. Estimated ~400 LoC decompiler / ~900 LoC total, 3–5 dev-days. Soft prereqs: F-ircore-reflected-property-emit, R-ir-grammar-harmonize, F-ir-authoring-guide.
- `#2-dual-surface-mandate` `OPEN` reporter 2026-05-13 — Locking in the **dual-surface invariant** before implementation. BOTH surfaces are MANDATORY, neither is optional: (a) asset-dump sidecar `msir.txt` registered in `AssetDumpHandler.h` `DumpFileNames` (slotted between MetaSound JSON dump and AGIR) and emitted from the MetaSound dispatch branch in `AssetDumpHandler.cpp`, written under `.editor-automation/asset-dumps/.../<asset>/msir.txt`; (b) MCP RPC `audio.authoring.decompile_metasound` handler at `Private/Handlers/Audio/MetaSound/MetaSoundDecompileHandler.cpp`, callable via `call("audio.authoring.decompile_metasound", { assetPath })`, returns `{ ir, warnings }`. Both surfaces MUST share ONE builder function `BuildMetaSoundIrText(UObject* MetaSoundAsset) → FIrResult { Text, Warnings, bSuccess }` living in `Private/MSIR/MSIRDecompiler.cpp` (operating on the `IMetaSoundDocumentInterface` of the asset). The sidecar pipeline calls it for dump emission; the RPC dispatch calls it for direct response. Zero divergence between the two — never ship one without the other, never let the two implementations drift. If the dump branch needs format differences, route them through builder options, not a parallel implementation.
- `#3-implementation` `IN-REVIEW` developer — Implemented MSIRDecompiler + MetaSoundDecompileHandler. Files: Private/MSIR/{MSIRTypes.h,MSIRDecompiler.h,MSIRDecompiler.cpp}, Private/Handlers/Audio/MetaSound/MetaSoundDecompileHandler.cpp. Registered audio.authoring.decompile_metasound RPC plus IrSidecarRegistry entries for UMetaSoundPatch/UMetaSoundSource. Added DumpFileNames::Msir in AssetDumpHandler.h and msir.txt sidecar wires through the existing registry (no AssetDumpHandler.cpp dispatch edit needed). Wiki docs/wiki/audio.authoring.md replaces the "MetaSound has no text IR" claim with an MSIR section + decompile_metasound entry. Regression tests in Private/Tests/Assets/TestMSIRDecompiler.cpp cover header/section emission and dual-surface zero-divergence (RPC text vs msir.txt sidecar contents byte-exact).
- `#4-verify-dual-surface` `DONE` tester — Verified: `audio.authoring.decompile_metasound` on `/Game/Audio/MetaSounds/sfx_RandomStereo_nl_meta.sfx_RandomStereo_nl_meta` returned MSIR text with interfaces, inputs, nodes, wires, and `warnings: []`; `asset.dump` to `C:/tmp/msir-verify-F-metasound-decompile-msir` wrote `msir.txt`, and the sidecar contained the same MSIR text form.
