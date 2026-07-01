---
id: B-asset-dump-agir-missing-for-child-animbp
title: "asset.dump silently drops agir.txt for child AnimBPs (no own anim graphs, parent ABP_*_C)"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, agir, animgraph, inheritance]
---

# asset.dump silently drops agir.txt for child AnimBPs

Child `UAnimBlueprint` assets — AnimBPs whose `parentClass` is another
`ABP_*_C` rather than `/Script/Engine.AnimInstance` — never produce an
`agir.txt` sidecar in their `asset.dump` folder, while every base
AnimBP in the same corpus does. The asset dump for these child BPs
therefore carries no AGIR record at all, even though they typically
override anim-node defaults (BlendSpace asset refs, sequence refs,
flags) inherited from the parent's anim graph — and those overrides
are visible in `properties.json`.

## Repro

Inspect dumps under `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/`:

- `App/ThirdPerson/Characters/Animations/ABP_Quinn/` — missing `agir.txt`.
  `meta.json` shows `parentClass = "/App/ThirdPerson/Characters/Animations/ABP_Manny.ABP_Manny_C"`.
  `properties.json` shows ~3 overridden anim-node fields inherited from
  `ABP_Manny_C` (`AnimGraphNode_BlendSpacePlayer`, `AnimGraphNode_SequencePlayer_3`).
- `App/ThirdPerson/Characters/Animations/ABP_Manny/` — has `agir.txt`
  (full Locomotion state machine, cached poses, etc.). `parentClass` is
  `/Script/Engine.AnimInstance`.
- Same pattern for `ABP_Manny_PostProcess` (has agir) vs `ABP_Quinn_PostProcess`
  (also has agir — its parentClass is `AnimInstance`, not a child).

A `find` across the dump corpus shows **12 child AnimBPs** all missing
`agir.txt`, including every per-weapon LinkedLayer override
(`ABP_PistolAnimLayers`, `ABP_RifleAnimLayers`, `ABP_ShotgunAnimLayers`,
`ABP_UnarmedAnimLayers` + each `_Feminine` variant), `ABP_Mannequin_TopDown`,
and three demo-fork copies of `ABP_Quinn`. So this is the default
Lyra/Lyra-fork inheritance pattern, not a one-off.

## Root cause

`FAGIRDecompiler::Decompile` (`Private/AGIR/AGIRDecompiler.cpp:79-193`)
walks `AnimBlueprint->FunctionGraphs` filtered by `IsTopLevelAnimGraph`
plus interface-reachable layer graphs. A child AnimBP that doesn't
redefine its own AnimGraph contributes zero entries to `TopLevelGraphs`.
The branch at line 175 (`TopLevelGraphs.Num() == 0`) returns success
with empty `AGIRText` and a single warning string `"Anim BP has no
anim graphs."`.

`AssetDumpHandler::CollectIrSidecars` (`AssetDumpHandler.cpp:329-360`)
then runs the empty-text guard at line 356 (`if (!Result.Text.IsEmpty())`)
and silently drops the sidecar — no diagnostic, no stub file. The
`anim_graph.json` aspect runs through the same path-shaped branch in
`BuildAnimGraphJson` and emits `{pages:[], state_machines:[],
anim_node_classes:[]}`, which at least keeps the file present but
empty; AGIR has no analogous fallback.

The result is asymmetric and consumer-hostile: tooling that walks the
dump tree expects every AnimBP folder to carry the canonical aspect
set, can't distinguish "AGIR genuinely failed" from "AGIR was
intentionally skipped" from "this AnimBP delegates its graph", and
loses any record that the child exists in AGIR terms.

## Recommended behavior

Emit `agir.txt` for every AnimBP, including children with no
top-level graphs. Two reasonable shapes:

1. **Minimal stub** — when the BP has no own anim graphs but a
   `parentClass` that is itself a `UAnimBlueprintGeneratedClass`, emit
   a single header line such as:

   ```
   # ==== AnimBP delegates to parent ====
   delegates_to `/App/ThirdPerson/Characters/Animations/ABP_Manny.ABP_Manny_C`
   ```

   This is enough for a consumer to know the child exists in AGIR
   terms and where to look for the actual graph.

2. **Child overrides** — additionally enumerate inherited anim-node
   field overrides (already available in `properties.json` as
   `inherited_from + is_overridden_locally` pairs on `AnimGraphNode_*`
   keys) as a `node_overrides { ... }` block. This is the AGIR-level
   counterpart of `properties.json` and gives the round-trip story a
   place to grow.

Option 1 is the minimum viable fix; option 2 is the "complete the
picture" extension. Either way the sidecar registry's empty-text guard
should not silently drop a successful decompile for a recognised asset
type — at minimum, an `AGIR_NO_GRAPHS` diagnostic should land in the
file-errors map so consumers see something happened.

**Fix:** In `FAGIRDecompiler::Decompile`, when `TopLevelGraphs.Num() == 0`
and `AnimBlueprint->ParentClass` is a `UAnimBlueprintGeneratedClass`,
emit the `delegates_to` stub into `AGIRText` instead of leaving it
empty. Optionally walk the CDO's `AnimGraphNode_*` properties (or
`properties.json` overrides) to emit a `node_overrides` block. The
`if (!Result.Text.IsEmpty())` guard at `AssetDumpHandler.cpp:356` is
then satisfied without needing to change the sidecar pipeline.

**Workaround:** Consumers can look at `properties.json` for `inherited_from`
entries on `AnimGraphNode_*` keys to discover overridden anim nodes,
and read `meta.json` `parentClass` to find the parent AGIR. There's
no in-AGIR record of the child's existence today.

## History
- `#1-initial-repro` `OPEN` reporter — Found while auditing the dump corpus. `ABP_Quinn` (parentClass = `ABP_Manny_C`) and 11 other child AnimBPs all missing `agir.txt` while every base AnimBP has one. Root-caused to the `TopLevelGraphs.Num() == 0` branch in `FAGIRDecompiler::Decompile` (`Private/AGIR/AGIRDecompiler.cpp:175-183`) returning success with empty `AGIRText`, combined with the empty-text guard in `AssetDumpHandler::CollectIrSidecars` (`AssetDumpHandler.cpp:356`) dropping the file. Recommend emitting a `delegates_to <parent>` stub for child AnimBPs (and optionally a `node_overrides` block from the CDO) so every AnimBP folder carries an AGIR record.
- `#2-agir-delegates-to-stub` `IN-REVIEW` developer — Implemented Option 1 (minimal `delegates_to` stub) in `AGIRDecompiler.cpp` empty-graphs branch: when `AnimBlueprint->ParentClass` casts to `UAnimBlueprintGeneratedClass`, emit a single `# ==== AnimBP delegates to parent ====` block with backtick-quoted parent path so `Result.AGIRText` is non-empty and the handler's empty-text guard passes. Added regression test `FAGIRChildAnimBPDelegatesToParentTest` loading `ABP_Mannequin_TopDown` and asserting `delegates_to` + parent classpath surfaces. Option 2 (node_overrides from CDO) deferred.
- `#3-verify-fix` `DONE` tester — Verified: `asset.dump` on `/App/ThirdPerson/Characters/Animations/ABP_Quinn` now writes `agir.txt` (listed in `writtenPaths`); contents are exactly the two-line stub `# ==== AnimBP delegates to parent ====` / ``delegates_to `/App/ThirdPerson/Characters/Animations/ABP_Manny.ABP_Manny_C` ``, matching the parentClass from `meta.json`. The empty-text guard at `AssetDumpHandler.cpp:356` is now satisfied for child AnimBPs.
- `#4-reverify-and-close` `DONE` tester — Re-verified after frontmatter was left as `IN-REVIEW`: ran `asset.dump` on `/App/ThirdPerson/Characters/Animations/ABP_Quinn`, `writtenPaths` includes `agir.txt`, file contents match the expected two-line `delegates_to` stub pointing at `ABP_Manny_C`. Promoted frontmatter `status` to `DONE`.
