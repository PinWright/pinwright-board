---
id: B-layout-engine-ba-derived
title: "PinWright's BPIR node layout engine self-describes as a port of the proprietary Blueprint Assist formatter and mirrors its names and structure, shipping under MIT without any notice; it must be replaced by PinWright's own implementation"
status: DONE
severity: High
category: bug
tags: [layout, licensing, bpir, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# PinWright's BPIR node layout engine is BA-derived and must be replaced

**Priority: P0.** PinWright's current Blueprint node layout engine is derived from
the Blueprint Assist (BA) marketplace plugin and must be replaced.

- **The engine.** It lives in `Source/PinWright/Private/Compiler/NodeLayoutEngine.{h,cpp}`
  and `Source/PinWright/Private/Compiler/NodeLayoutParameterFormatter.{h,cpp}`.
  `RunLayoutPass` in `Source/PinWright/Private/Compiler/BpirCompiler.cpp` calls it
  after every BPIR compile or insert.
- **The derivation was self-declared.** Before the reference cleanup, the source
  comments, `docs/bpir-compiler-internals.md` and `docs/lessons.md` described the
  engine as a port of BA's formatter. They cited BA source files and line numbers,
  and `lessons.md` told agents to read BA source before changing layout behaviour.
- **Names and structure match BA.** The engine mirrors BA's names and pass
  structure.
- **BA's licence.** BA is a paid Fab/Marketplace product. Its source headers read
  "All Rights Reserved" and it ships no redistribution licence.
- **PinWright's licence.** PinWright is published under MIT. `THIRD_PARTY_NOTICES.md`
  carries no notice for BA.

A public MIT repository whose code describes itself as a port of proprietary code,
and mirrors that code's names and structure, is a licensing exposure. The
exposure is independent of how well the code works.

## Decision (user, 2026-09-30)

1. **Replace the engine.** The current engine is replaced with PinWright's own
   implementation, `F-graph-layout-core`.
2. **Copying policy for the rewrite (user clarification, 2026-09-30).** Reading
   Blueprint Assist source, and reading PinWright's current layout code, is
   allowed. What is not allowed is direct copying without change: no verbatim or
   near-verbatim code, identifiers or structure. The new engine uses its own names,
   decomposition and algorithm, as specified in `F-graph-layout-core`. It must not
   carry over the current engine's function and type names or its pass structure.
3. **Remove the references.** Every Blueprint Assist reference is removed from the
   plugin repo: source comments, docs, the `lessons.md` instruction, and test
   comments. A separate agent is doing this now, as its own change.

## Acceptance criteria

- `rg -i "blueprint ?assist|BA's|BA formula|fpwong"` over `Plugins/PinWright`
  (source, docs, tests, `wiki-src`) returns nothing.
- The BA-derived files are deleted once `F-graph-layout-core` replaces the call
  site in `BpirCompiler.cpp`. No code path still reaches them.
- The replacement shares no verbatim or near-verbatim code, identifiers or pass
  structure with Blueprint Assist or with the deleted engine. The reviewer confirms
  this in the `IN-REVIEW` entry.

## Severity justification

**High, worked as P0.** The rubric has no licensing class. The impact is legal
exposure on every public release, not a runtime defect, and it outranks the
functional layout work, which depends on this rewrite anyway.

## History
- `#1-initial-report` `OPEN` reporter — Gap analysis 2026-09-30: the BPIR layout engine self-describes as a port of the proprietary Blueprint Assist formatter and mirrors its names and structure, with BA file:line citations in docs and an instruction in lessons.md to read BA source; PinWright is MIT and carries no notice. User decision recorded: replace it with PinWright's own implementation (F-graph-layout-core) under the policy "reading BA source and the current code is fine; no direct copying without change — no verbatim/near-verbatim code, identifiers or structure"; all BA references removed from the plugin repo (separate agent, in progress).
- `#2-engine-replaced` `IN-REVIEW` developer — Replaced by F-graph-layout-core (`Layout/PwGraphLayout*`): `RunLayoutPass` in `Compiler/BpirCompiler.cpp` now calls `PwGraphLayout::ArrangeEdGraph`; `Compiler/NodeLayoutEngine.{h,cpp}`, `Compiler/NodeLayoutParameterFormatter.{h,cpp}` and `Tests/Bpir/TestNodeLayout.cpp` are deleted, and no code path or include reaches them (`rg NodeLayoutEngine|NodeLayoutParameterFormatter|BpirLayout::` over `Plugins/PinWright` is empty). The size estimator was rewritten in `Layout/BlueprintNodeSizeAdapter.cpp`; the settings knobs tied to the old passes were renamed or removed. The last Blueprint Assist mention (`docs/lessons.md`) was reworded and the old pass names were removed from `docs/bpir-compiler-internals.md`, `docs/bpir-test-matrix.md`, `docs/test-organization.md` and test comments; `rg -i "blueprint ?assist|BA's|BA formula|fpwong"` over `Plugins/PinWright` (source, docs, tests, wiki-src) returns nothing. The implementer did not consult Blueprint Assist source; the replacement shares no identifiers or pass structure with the deleted engine (see F-graph-layout-core `#3`) — reviewer to confirm. Tests: `PinWright.layout.core.{LinearChain,Branch,Sequence,DiamondReconvergence,ExecLoop,ThreeEvents,SharedDataNode,DataChainThreeDeep,MaterialMathChain,AnimPoseChain,FixedObstacleAvoided,MetricsScoreBrokenLayoutsWorse}`, `PinWright.layout.blueprint.{EventGraphFixture,CompilePassArrangesEveryEntry}`, `PinWright.layout.material.GrowsLeftFromOutput`, `PinWright.layout.anim.PoseChainGrowsLeftFromResult`, `PinWright.layout.controlrig.DataChainFlowsRight`; filter `PinWright.layout` (also covers the existing `layout.metrics.*`).
- `#3-clean-room-review` `IN-REVIEW` reviewer — **Verdict: CONFIRMED clean. No must-fix items.** Scope: the replacement at plugin HEAD (`0a3b3acc`) was compared with the deleted engine at `5e99e947`. The deleted engine is used as the proxy for Blueprint Assist, whose source was not available and was not fetched. The pre-cleanup tree (`8748c637`) was read only to see which parts the old code itself attributed to BA. Those parts are `GetChildX`/FormatX, `FormatY_Recursive` and its collision loop, same-row marking, and the parameter formatter. The size estimator was never attributed to BA. **Method and numbers.** (1) Token similarity used comments and preprocessor lines stripped, with maximal common runs seeded from 8-token shingles. In raw mode (identifiers kept), the BA-derived files (`NodeLayoutEngine.*`, `NodeLayoutParameterFormatter.*`, `TestNodeLayout.cpp`) share 97 distinct runs with all new layout sources and tests. The longest is 23 tokens, a test-fixture idiom (`BP->UbergraphPages.Num() > 0 ? ... : nullptr`). The longest in non-test code is 16 tokens (`for (const UEdGraphPin* Pin : Node->Pins) { if (!`). `PwGraphLayout.{h,cpp}` against the old engine gives a longest run of 8 tokens and covers 0.3 % of the new tokens. Every raw run inspected is a UE API idiom (`GetFontMeasureService`, `GetNodeTitle(ENodeTitleType::ListView)`, `GetDefault<UBpirLayoutSettings>()`, `SpawnNode<...>(Graph, 0, 0)`). (2) In normalized mode (identifiers, numbers and strings abstracted), old engine against `PwGraphLayout.cpp` gives a longest run of 20 tokens. At 12-token shingles it covers 13.5 % of the new tokens. The unrelated control `AudioGen/PwMusicScore.cpp` gives 31 / 13.2 %, and `Handlers/Animation/SkeletonHandler.cpp` gives 22 / 8.1 %, so the core sits at noise level. Every normalized run of 14 tokens or more is a generic shape: for-loops, member declarations, a `(…, const UBpirLayoutSettings& Settings)` signature. (3) Identifier overlap, after dropping UE and engine API names: `UBpirLayoutSettings`, `bEnableBpirLayoutPass`, `PinRowHeightPx`, `HeaderHeightPx` and `HorizontalPaddingPx` are pre-existing PinWright settings. `EstimateNodeSize` is the pre-existing `INodeSizeAdapter` seam method in `GraphLayoutMetrics.h`. `ColumnGapPx` was a private 40 px estimator constant in the old code and is now an 80 px flow-gap setting, a coincidental reuse with a different meaning. The rest are generic locals: `Anchor`, `Consumer`, `Cursor`, `Head`, `Queue`, `Child`, `Offset`, `Key`, `Grid`, and `FromPin`/`ToPin` as wire fields. None of the old engine's own names survive in source, tests, docs or wiki-src: FormatX/FormatY, FFormatXInfo(Map), FPinLink, GetChildX, GetPinsOfSameHeight, FClusterBoundsRegistry, FormatParameterNodes, ResolveConsumerClusterOverlaps, ResetRelativeToAnchor, SnapToGrid, the old setting names. The only exception is the CHANGELOG migration note for the renamed settings, which is legitimate. (4) Pass structure: the old pipeline was single anchor, dual-stack BFS walk, FormatX ×2 around the parameter formatter, same-row marking, recursive FormatY with a capped push-down loop, a cluster-overlap BFS, translation back to the anchor, then a directional SnapToGrid. The new pipeline is multi-root DFS forest with back-edge skip, first-consumer claim, per-block longest-path levels with 3 barycenter sweeps, one topological X pass, preorder Y with an interval-sweep first fit over the exec spine, then commit. The decomposition and control flow differ. (5) `rg -i "blueprint ?assist|BA's|BA formula|fpwong"` over the whole plugin (source, docs, tests, wiki-src, CHANGELOG) returns nothing. (6) No include, Build.cs, doc or test still refers to the deleted files. The full findings are in F-graph-layout-core `#6-clean-room-review`.
- `#7-verified-done` `DONE` tester — Acceptance checked: (1) `rg -i "blueprint ?assist|BA's|BA formula|fpwong"` over `Plugins/PinWright` returns nothing (re-run by the reviewer, `#3-clean-room-review`); (2) the BA-derived files are deleted and no include, build rule, doc or test reaches them; (3) the independent reviewer found no verbatim/near-verbatim code, identifiers or pass structure shared with the deleted engine (longest shared normalized run 16 tokens, a stock UE pin loop; the new core shares at most 8). Replacement verified by the round-6 scoped run `636bc41c` (offscreen, Linux Vulkan, UE 5.8; 1183/1183, every skip a host `fixture-missing` marker outside layout) at PinWright `9bb70b90`. The BA-derived engine remains in pre-1.0 git history; the owner accepted that.
