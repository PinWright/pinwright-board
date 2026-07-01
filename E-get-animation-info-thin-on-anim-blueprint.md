---
id: E-get-animation-info-thin-on-anim-blueprint
title: "`animation.authoring.get_animation_info` on an AnimBlueprint emits only {assetType, skeletonPath, parentClass} — no AnimGraph state machines / states / transitions / slot nodes / coords, so the build round-trip is unverifiable without asset.dump (the sole un-fixed branch of the get_animation_info parity family)"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [animation, anim-blueprint, anim-graph, asset-dump, parity, get-animation-info, agir, state-machine, slot-node, docs]
---

# `get_animation_info` on an AnimBlueprint is metadata-only — the AnimGraph (state machines, slot nodes, coords) lives only in the asset.dump sidecars

`get_animation_info` is documented as the introspection dual used to confirm
an asset's shape after authoring (the *Inspect-after-mutate* loop on the
`animation.authoring` wiki overlay). Its sibling branches have all been
extended to deliver full readback parity by delegating to the matching dump
builder:

- `UAnimSequence` — rich shape (`E-rpc-animation-extend-get-animation-info`, DONE).
- `UAnimMontage` — `JsonBuilders::MergeMissingFields(AnimInfo, AnimMontageDumpBuilder::BuildAnimMontageJson(Montage))` (`AnimationAuthoringHandler_Sequence.cpp:1852`), so sections / links / slots round-trip live.
- `UBlendSpace` — `JsonBuilders::MergeMissingFields(AnimInfo, BlendSpaceDumpBuilder::BuildBlendSpaceJson(BlendSpace))` (`AnimationAuthoringHandler_Sequence.cpp:1868`), so samples / axes round-trip live.

The **`UAnimBlueprint` branch is the one branch in that family that was never
extended.** `AnimationAuthoringHandler_Sequence.cpp:1870-1878` emits only three
metadata fields and never merges the AnimGraph dump:

```cpp
else if (UAnimBlueprint* AnimBP = Cast<UAnimBlueprint>(Asset))
{
    AnimInfo->SetStringField(TEXT("assetType"), TEXT("AnimBlueprint"));
    if (AnimBP->TargetSkeleton)
    {
        AnimInfo->SetStringField(TEXT("skeletonPath"), AnimBP->TargetSkeleton->GetPathName());
    }
    AnimInfo->SetStringField(TEXT("parentClass"), AnimBP->ParentClass ? AnimBP->ParentClass->GetName() : TEXT(""));
}
```

So after the canonical AnimBP build-out (`create_anim_blueprint` →
`add_state_machine` → `add_state` ×N → `add_transition` ×N →
`add_slot_node`), `get_animation_info` confirms only the **skeleton binding
and parent class** — never *which* state machine exists, *which* states /
transitions it holds, *which* slot node / group landed, or the graph **node
coordinates** the task asked to verify. The only way to confirm the AnimGraph
structure is to fall back to the read-only `asset.dump` sidecars
(`anim_graph.json` for the structured state-machine/transition shape, and
`agir.txt` for the slot-node name + node coordinates), which
`AnimGraphDumpBuilder::BuildAnimGraphJson(UAnimBlueprint*)`
(`Private/Handlers/Animation/AnimGraphDumpBuilder.cpp:299`, already
`PINWRIGHT_API`-exported and used by `asset.dump` at
`AssetDumpHandler.cpp:643`) emits with full fidelity. The state-machine /
slot-node / coordinate detail is therefore **exclusive to the dump** —
unreachable through the live introspection RPC. This is the same
no-exclusive-dump-fields parity class as DONE
`E-rpc-animation-extend-get-animation-info` (AnimSequence) and OPEN
`E-get-animation-info-thin-on-montage` / `E-get-animation-info-thin-on-blend-space`
(Montage / BlendSpace) — here repeated for the AnimBlueprint branch, the last
one not yet brought to parity. The call is not wrong (the three fields are
accurate, no crash, valid JSON) — it is too thin to verify the very structure
`add_state_machine` / `add_state` / `add_transition` / `add_slot_node` author.

This is distinct from `B-asset-dump-agir-missing-for-child-animbp` (DONE) and
`E-anim-graph-json-omits-transition-logic-blend` (IN-REVIEW): those are
**dump-side** fidelity issues (a missing/incomplete sidecar). This ticket is the
**live-RPC parity** gap — even with both sidecars perfect, `get_animation_info`
on an AnimBP would still return only metadata, so the readback parity gap is
independent.

## Workaround
Read the `asset.dump` sidecars for the AnimBP: `anim_graph.json`
(`state_machines[]` with `states[]` / `transitions[]`) for the structured
graph, and `agir.txt` for the slot-node name + node coordinates.

## Fix
Additively extend the `UAnimBlueprint` branch of `get_animation_info` to
delegate to the dump builder, mirroring the Montage and BlendSpace branches
two cases above it:

```cpp
JsonBuilders::MergeMissingFields(AnimInfo, AnimGraphDumpBuilder::BuildAnimGraphJson(AnimBP));
```

`BuildAnimGraphJson` is already exported (`AnimGraphDumpBuilder.h:16`) and is
the symmetric counterpart of the builders the other two branches already call,
so this is a one-line, low-risk parity fix. Keep `assetType` / `skeletonPath` /
`parentClass` for backward compatibility (they take precedence under
`MergeMissingFields`). Separately (downstream wiki process, not this audit's
job), document on `docs/wiki-src/animation.authoring.md` that the AnimBP
readback's structured AnimGraph shape comes through `get_animation_info`
(post-fix) and that slot-node *coordinates* surface in the `agir.txt` sidecar.

## History
- `#2-already-fixed` `IN-REVIEW` developer — Already resolved in current source; no code written. The exact one-line `**Fix:**` the ticket proposes is already committed at HEAD: `Source/PinWright/Private/Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp:1895` reads `JsonBuilders::MergeMissingFields(AnimInfo, AnimGraphDumpBuilder::BuildAnimGraphJson(AnimBP));` inside the `UAnimBlueprint` branch (cast at :1879), with the supporting `#include "Handlers/Animation/AnimGraphDumpBuilder.h"` at :15. `git status` shows the handler clean (committed, not a working-tree edit); it landed via commit `a616fd1` ("E-get-animation-info-thin-on-anim-blueprint: All three lenses voted valid…"), which also shipped the regression test `FAuthoringGetAnimationInfoAnimBlueprintParityFieldsTest` / `PinWright.animation.authoring.get_animation_info.AnimBlueprintParityFields` (`Tests/Gameplay/TestAnimationHandlers.cpp:2111`) asserting the merged `state_machines[]` shape. The ticket body's quoted "metadata-only" block (lines 28-38) no longer matches reality. The AnimBP branch at :1895 completes the get_animation_info parity family (AnimSequence/Montage/BlendSpace already merge). The prior fix commit only added the lease fields to frontmatter and never flipped the status, leaving it OPEN and re-claimed by fuzz3; flipping OPEN -> IN-REVIEW and releasing the claim so a tester verifies. All three validity lenses concur (already-fixed / wontfix / regression-already-fixed).
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `ABP_DinoDragon_Locomotion` build-out task (namespace `animation.authoring`, focus `animation.authoring.add_slot_node`, outcome clean — every RPC succeeded first try). Friction note (verbatim): *"get_animation_info alone doesn't show AnimGraph node detail/coords so asset.dump's agir.txt was needed for the slot-name + coordinate readback, which is reasonable."* The reporter deemed it reasonable, but it is the same documented no-exclusive-dump-fields parity class the family already fixed for three of four anim asset kinds. Source-confirmed: `AnimationAuthoringHandler_Sequence.cpp:1870-1878` (UAnimBlueprint branch sets only `assetType`/`skeletonPath`/`parentClass`) versus the Montage branch (line 1852) and BlendSpace branch (line 1868) which both already `JsonBuilders::MergeMissingFields(...)` their dump builder; `AnimGraphDumpBuilder::BuildAnimGraphJson(UAnimBlueprint*)` is already `PINWRIGHT_API`-exported (`AnimGraphDumpBuilder.h:16`, impl `AnimGraphDumpBuilder.cpp:299`) and consumed by `asset.dump` (`AssetDumpHandler.cpp:643`), so the one-line delegation is available. Distinct from the dump-side tickets `B-asset-dump-agir-missing-for-child-animbp` (DONE) and `E-anim-graph-json-omits-transition-logic-blend` (IN-REVIEW). Call-log: 13 calls; `get_animation_info` ×1 (returned metadata only) then forced an `asset.dump` (anim_graph.json + agir.txt) to obtain the slot-name + state-machine + coordinate verification the task's step-6 sanity check required. Tagged docs for the downstream `animation.authoring.md` overlay note.
