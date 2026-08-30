---
id: F-pcg-decompile-ir
title: "PCG graph decompile to text IR (read-only)"
status: DONE
severity: Medium
category: feature
tags: [pcg, ir, decompile, rpc, read-only]
---

# PCG Graph Decompile to Text IR (Read-Only)

Add `pcg.decompile` — a read-only text IR emitter for `UPCGGraph` assets
analogous to `material.decompile_mgir`, `blueprint.decompile_bpir`, and
`animation.decompile_agir`. Scope is **decompile direction only**: reading
existing PCG graphs as text for inspection, dump parity, and diffing. The
compile direction (`pcg.compile_pcgir`) is **out of scope** for this
ticket and is deferred to a future `F-pcg-compile-ir` ticket if read-side
demand surfaces compile-side friction.

## Why decompile-first

The MGIR-after-Niagara precedent and the BPIR/AGIR rollout both shipped
their decompile direction either alongside or just before the compile
direction, and in every case decompile is what users reached for first —
inspecting an unknown asset, diffing before/after an edit, sanity-checking
a generated graph. PCG is a node-graph surface with the same shape and
benefits from the same read path. Decoupling decompile from compile here
keeps scope small, lets the grammar evolve based on real read traffic
before being locked in by a parser, and ships the highest-leverage half
of the work first.

`F-pcg-core-graph` (the imperative authoring sibling) gives agents
`pcg.add_node` / `pcg.connect_pins` / `pcg.inspect` — fine for *writing*
graphs and for structured JSON reads. `pcg.decompile` is the **text**
reader companion: dump-parity sidecars, human-diffable snapshots, and
single-shot inspection of large graphs where JSON inspection output is
too verbose to skim.

## Output shape

Mirror the existing IR conventions (MGIR/BPIR/AGIR) rather than inventing
a new grammar shape:

- **Entry block** naming the target asset:
  `entry pcg "/Game/PCG/G_FoliageScatter" { ... }`
- **Node statements** keyed by class path with a stable local id and
  position annotation:
  `node N1 = "/Script/PCG.PCGSurfaceSamplerSettings" @(x, y) { ... }`
- **Property block** inside each node, one reflected property per line,
  matching the MGIR property syntax (literals, vectors, asset refs, enum
  names by symbolic name not raw integer per `F-decompile-enum-names`).
- **Edge statements** at graph scope addressing nodes by local id and
  pin name:
  `connect N1.OutPoints -> N2.InPoints`
- **Subgraph references** emitted as plain node statements pointing at
  `UPCGSubgraphSettings` with the referenced `UPCGGraph` asset path in
  the property block — flatten on decompile, do not recurse into the
  referenced graph (matches the MGIR composite-flatten policy).
- **Node positions** preserved on decompile; logical-equivalence rules
  from `feedback_ir_logical_not_visual` still apply (comments, colors,
  node ordering in the editor list are out of scope).

Section headers, indentation, and tokenizer choices should reuse the
shared `IrCore` foundations the MGIR rollout introduced (`F-ircore-
shared-text-helpers`) rather than fork a parallel lexer.

## Use cases

1. **Dump parity sidecar.** `asset.dump_folder` already emits `bpir.txt`,
   `mgir.txt`, `agir.txt` next to per-asset metadata under
   `.editor-automation/asset-dumps/`. PCG graphs currently dump only
   their JSON properties block. Adding `pcgir.txt` to the dump pipeline
   (gated on `UPCGGraph` class) gives the offline corpus a uniform
   text-readable layer for every node-graph asset class.
2. **Agent inspection of unknown PCG graphs.** Reading a 30-node PCG
   graph as MGIR-shaped text is materially faster than walking the JSON
   inspect output node-by-node.
3. **Diffing before / after edits.** Once `F-pcg-core-graph` lands and
   agents start mutating PCG graphs, a text decompile gives reviewers a
   `git diff`-style readout of what changed without needing a custom
   structural differ.
4. **Code review on generated content.** Procedurally generated PCG
   graphs (e.g. asset-bake pipelines) can be sanity-checked by
   decompiling and reading the IR rather than opening the editor.

## Compile direction is out of scope

`pcg.compile_pcgir` is **explicitly deferred**. The grammar above is a
sketch for the emitter only — the parser, type-spec resolver, layout
pass, and round-trip test matrix that a compile direction would require
are a separate, larger body of work. File a follow-on `F-pcg-compile-ir`
ticket only if real read-direction usage surfaces friction that compile
would relieve (e.g. agents repeatedly hand-editing decompiled `pcgir.txt`
and feeding it back through `pcg.add_node` chains). The MGIR-after-
Niagara rollout shows this ordering works: ship the reader, let the
grammar harden against real traffic, then build the writer if demand
justifies it.

## Cross-references

- [`F-pcg-core-graph`](F-pcg-core-graph.md) — imperative authoring
  sibling; ships `pcg.create_graph` / `pcg.add_node` / `pcg.connect_pins`
  / `pcg.inspect`. Decompile depends on the same `pcg.*` namespace
  scaffolding (module init, `PCG` + `PCGEditor` build dependencies) and
  should land after or alongside it.
- [`F-pcg-filters-and-subgraphs`](F-pcg-filters-and-subgraphs.md) —
  typed-helper follow-on; decompile must emit subgraph nodes flatly
  regardless of whether they were created via the generic `add_node` or
  the typed `add_subgraph`.
- [`F-mgir-material-graph-ir`](F-mgir-material-graph-ir.md) — closest
  prior art for grammar shape, decompile-only flattening, and round-trip
  semantics.
- [`F-ircore-shared-text-helpers`](F-ircore-shared-text-helpers.md) —
  reuse the shared tokenizer/emitter primitives rather than forking.
- [`F-decompile-enum-names`](F-decompile-enum-names.md) — symbolic enum
  names on decompile, not raw integers; applies to PCG enum-typed
  properties too.
- [`F-asset-dump-text-mirror`](F-asset-dump-text-mirror.md) — pipeline
  for routing `pcgir.txt` into the dump sidecar layout.

**Fix:** Add `pcg.decompile(assetPath, [includeReferencedSubgraphs])` under `Source/PinWrightPCG/Private/Handlers/PCG/PCGDecompileHandler.cpp` (creates the `Handlers/PCG/` directory). Implement `FPCGIRDecompiler::DecompileGraph(UPCGGraph*, FPCGIRDecompileOptions)` at `Private/PCGIR/PCGIRDecompiler.{h,cpp}` mirroring `FMGIRDecompiler::DecompileMaterial`. Wire dual-surface dispatch via the existing `REGISTER_DECOMPILE_IR(...)` macro in `Private/Utils/IrSidecarRegistry.h` — registration drives both the dump-sidecar pipeline (`AssetDumpHandler.cpp:328` iterates the registry) and the RPC handler (calls the decompiler directly). Add `DumpFileNames::PcgIr = TEXT("pcgir.txt")` to `AssetDumpHandler.h`. Reuse `Private/IrCore/IrTextUtils.{h,cpp}` for quoting, position formatting, and the reflected-property walk — F-ircore-shared-text-helpers has shipped. Add `PCG` to `PinWright.Build.cs` via `TryAddConditionalModule(...)`; `PCGEditor` is not needed for read-only decompile. Add 6 decompile-side tests under `Private/Tests/Assets/TestPCGIRDecompile_*.cpp` (empty graph, single-node, multi-node + edges, subgraph flatten, enum symbolic names, asset-ref properties) plus one sidecar-registration test. Add `docs/wiki/pcg.pcgir.md` mirroring `material.mgir.md`.

## History
- `#1-decompile-only-scope` `OPEN` reporter — Filed `pcg.decompile` as the read-direction companion to `F-pcg-core-graph`'s imperative authoring core. Verified zero PCG coverage in `rpc-method-reference.generated.md` and `Source/EditorAutomationRpcGateway/Private/Handlers/`. Scoped to decompile only — compile direction (`pcg.compile_pcgir`) deferred to a future `F-pcg-compile-ir` ticket pending read-side demand, mirroring the MGIR-after-Niagara rollout pattern where decompile shipped first and let grammar harden against real read traffic before compile was committed to. Grammar sketch reuses MGIR/BPIR/AGIR conventions (entry block, class-path-keyed node statements with `@(x, y)` position, property block, edge statements by local id + pin name, subgraph flattening on decompile). Primary use cases: `pcgir.txt` sidecar in `asset-dumps/`, agent inspection, before/after diff readability, code review on generated graphs. Cross-refs to `F-pcg-core-graph`, `F-pcg-filters-and-subgraphs`, `F-mgir-material-graph-ir`, `F-ircore-shared-text-helpers`, `F-decompile-enum-names`, `F-asset-dump-text-mirror`.
- `#2-dual-surface-mandate` `OPEN` reporter 2026-05-13 — Locking in the **dual-surface invariant** before implementation. BOTH surfaces are MANDATORY, neither is optional: (a) asset-dump sidecar `pcgir.txt` registered in `AssetDumpHandler.h` `DumpFileNames` and emitted from the `UPCGGraph` dispatch branch in `AssetDumpHandler.cpp`, written under `.editor-automation/asset-dumps/.../<asset>/pcgir.txt` next to `meta.json`; (b) MCP RPC `pcg.decompile` handler at `Private/Handlers/PCG/PCGDecompileHandler.cpp`, callable via `call("pcg.decompile", { assetPath, includeReferencedSubgraphs? })`, returns `{ ir, warnings }`. Both surfaces MUST share ONE builder function `BuildPcgIrText(UPCGGraph*, FPcgIrBuildOptions) → FIrResult { Text, Warnings, bSuccess }` living in `Private/PCGIR/PCGIRDecompiler.cpp`. The sidecar pipeline calls it for dump emission (default options); the RPC dispatch calls it for direct response (caller-supplied options like `includeReferencedSubgraphs`). Zero divergence between the two — never ship one without the other, never let the two implementations drift. Option flags route through builder parameters, not a parallel implementation.
- `#3-pcg-decompile-ir-and-sidecar` `IN-REVIEW` developer — shipped pcg.decompile RPC + pcgir.txt sidecar; rewrote `**Fix:**` paragraph to reference existing `IrSidecarRegistry` pattern (history #2's bespoke `BuildPcgIrText` shape was misformulated — registry already enforces dual-surface). Added `PCGIRDecompiler` + `PCGDecompileHandler` + 7 tests + `pcg.pcgir.md` wiki overlay; added `PCG` conditional module to `Build.cs`; added `DumpFileNames::PcgIr`.
- `#4-verify-pcgir-dual-surface` `DONE` tester — Verified: `pcg.decompile` on `/PCG/UnitTest/UnitTestSubgraphCollapse` returned PCGIR text with `entry pcg`, terminal nodes, graph nodes, edges, subgraph properties, and empty warnings; `asset.dump` for the same asset wrote `C:/tmp/mcp-verify-F-pcg-decompile-ir/PCG/UnitTest/UnitTestSubgraphCollapse/pcgir.txt` with matching PCGIR entry text.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. Path(s) here that move by more than the prefix in this ticket, taken from the plugin's rename history rather than the prefix rule: `Source/EditorAutomationRpcGateway/Private/Handlers/PCG/PCGDecompileHandler.cpp` → `Source/PinWrightPCG/Private/Handlers/PCG/PCGDecompileHandler.cpp`. One file basename renamed by the same plugin commit is repointed with it (`EditorAutomationRpcGateway.Build.cs` → `PinWright.Build.cs`, `EditorAutomationRpcGateway_SCSHandlers` / `_BlueprintHandlers_List` → `PinWright_*`), verified present at HEAD. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
