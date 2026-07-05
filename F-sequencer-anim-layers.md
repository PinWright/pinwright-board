---
id: F-sequencer-anim-layers
title: "Sequencer non-destructive anim layers: create, reorder, merge/collapse"
status: OPEN
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
