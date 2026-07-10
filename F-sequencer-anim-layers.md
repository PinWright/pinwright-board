---
id: F-sequencer-anim-layers
title: "Sequencer non-destructive anim layers: create, reorder, merge/collapse"
status: WONTFIX
severity: Medium
category: feature
tags: [sequencer, animation, layers, parity-ue58]
---

# Sequencer non-destructive anim layers: create, reorder, merge/collapse

PinWright's sequencer namespace has no anim-layer operations (grep `layer|Layer` in `Source\PinWright\Private\Handlers\Sequencer\` finds nothing layer-related). Non-destructive layering (additive/override layers over a base performance, later merged) is a core cinematics-iteration workflow an agent currently cannot drive.

UE 5.8 parity evidence: AnimationAssistantToolset sequencer suite (`C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\AnimationAssistantToolset\Content\Python\animation_toolset\toolsets\sequencer.py`) exposes anim layer duplicate/reorder/merge/collapse operations among its 140 sequencer tools.

Note: existing board ticket `F-anim-layered-blend-bone-weights` (DONE) is AnimBP-side layered blend, unrelated to Sequencer layers.

Proposed scope:
- `sequencer.add_anim_layer(sequence, binding, mode)` - additive or override layer.
- `sequencer.list_anim_layers` / `sequencer.reorder_anim_layer` / `sequencer.set_layer_weight`.
- `sequencer.merge_anim_layers(sequence, binding, layers[])` - collapse to a single track.

Acceptance: base + additive layer keyed separately, weight change reflected in evaluated pose, merge produces a single section whose evaluation matches the pre-merge composite.

## History
- `#1-no-anim-layers` `OPEN` reporter — Sequencer namespace lacks non-destructive anim layers entirely; Epic 5.8 ships duplicate/reorder/merge/collapse layer ops. File create/reorder/weight/merge RPCs.
- `#2-headless-incompatible` `WONTFIX` developer — Declined as scoped (mis-scoped + architecturally incompatible with PinWright's headless MovieScene-direct Sequencer model; verified against UE 5.7 engine source). (1) EVERY engine anim-layer entry point — `UControlRigSequencerEditorLibrary` `AddAnimLayerFromSelection`/`Delete`/`Duplicate`/`MergeAnimLayersWithSettings`/`GetAnimLayers` AND `CollapseControlRigAnimLayers` (`ControlRigSequencerEditorLibrary.cpp:2962/3527/3550/3573/3616/3637`) — routes through `UAnimLayers::GetSequencerFromAsset()` (`AnimLayers.cpp:1454` = `GetCurrentLevelSequence()` + open-toolkit `FindEditorForAsset()`), which is null under headless automation, so all no-op ("Need open Sequencer"); none is `(sequence,binding)`-addressable as the ticket proposes (create is outliner-selection-driven with no binding arg; merge is by index on the active Sequencer). (2) `reorder_anim_layer` has NO engine function at all. (3) The per-layer section linkage (`UAnimLayer::AnimLayerItems`) is populated only by private, selection-driven engine code, and `UAnimLayer`/`UAnimLayers` are `MinimalAPI` + Private-header — so a FUNCTIONAL layer cannot be constructed or validated via reflection headless (a reflection fork of editor-only internals is fragile and out of scope). (4) `parity-ue58` framing is wrong — `MergeAnimLayers` is `UE_DEPRECATED(5.6)`; the API is present/complete on 5.7. (5) The "agent cannot drive layering at all" premise is false — additive layering already ships via `sequencer.add_controlrig_track(layered:true)` (creates an additive rig, `ControlRigSequencerHandler.cpp:392-399`) + `sequencer.key_controls` + `sequencer.get_control_value`. The full `UAnimLayers` workflow (formal layers, per-layer weight, merge/collapse, reorder) needs a live-`ISequencer` automation harness (or a large fork of private engine code) that PinWright does not have; re-file with that explicit prerequisite if later prioritized. Untracked red test `TestSequencerAnimLayers.cpp` removed (no-code disposition must not leave a permanently-failing test).
