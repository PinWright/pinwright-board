---
id: B-agir-variable-pin-args-silently-dropped
title: "anim.compile_agir silently discards `$Variable` arguments on anim-node data pins — reports nodesCreated with no warning, and the decompile shows the pins unwired"
status: IN-REVIEW
severity: High
category: bug
tags: [anim, agir, compile_agir, anim-blueprint, silent-drop, pin-binding, blend-space]
encounters: 1
lastSeen: 2026-09-02T20:10:00Z
---

# AGIR accepts `$Variable` on a data pin and throws it away

## Repro

Fresh Anim Blueprint on the UE5 Mannequin skeleton, one AGIR compile that wires a 2D locomotion
blend space and a rifle aim offset to Anim BP member variables (`Direction`, `Speed`, `AimYaw` all
exist on the Blueprint at compile time):

```
anim.compile_agir {context:"/Game/FPS/AI/ABP_Enemy", mode:"Replace", save:true, text:
"entry anim_graph AnimGraph {
    output %rotation_offset_blend_space_0
    %blend_space_0 = call `/Script/AnimGraph.AnimGraphNode_BlendSpacePlayer`(BlendSpace: \"...MM_BS_Locomotion_2D\", X: $Direction, Y: $Speed)
    %slot_0 = call `/Script/AnimGraph.AnimGraphNode_Slot`(Source: %blend_space_0)
    %rotation_offset_blend_space_0 = call `/Script/AnimGraph.AnimGraphNode_RotationOffsetBlendSpace`(BasePose: %slot_0, BlendSpace: \"...MM_AO_Rifle\", X: $AimYaw)
}"}
```

Response:

```
{"mode":"Replace","assetPath":"/Game/FPS/AI/ABP_Enemy.ABP_Enemy","blocksCompiled":1,"nodesCreated":3,"warnings":[]}
```

`warnings` is empty. Immediately decompiling the same asset shows the three pose links intact and
**every `$Variable` argument gone**:

```
%blend_space_player_0 = call `...AnimGraphNode_BlendSpacePlayer`(BlendSpace: "...MM_BS_Locomotion_2D") ...
%rotation_offset_blend_space_0 = call `...AnimGraphNode_RotationOffsetBlendSpace`(BasePose: %slot_0, BlendSpace: "...MM_AO_Rifle") ...
```

`blueprint.graph.get_nodes {graphName:"AnimGraph"}` confirms it from the other side: the
`BlendSpacePlayer` node's `X` and `Y` pins and the aim offset's `X` pin all report `"linkedTo":[]`.

## Why this is worse than a rejection

An unwired blend-space input is not a visible failure. The node keeps playing — at the blend
space's origin sample. A 2D locomotion blend space whose `Speed` axis never leaves 0 plays the idle
cell forever, so the character *animates*, looks superficially fine in a thumbnail, and simply never
reacts to movement. There is no compile error, no runtime warning, and no red pin in the editor.
The only way I found the loss was decompiling my own successful compile and reading it line by line.

Compare the surrounding behaviour: pose links (`Source:`, `BasePose:`) and asset arguments
(`BlendSpace:`) in the same argument list are honoured. Only the value pins vanish, so the failure
looks like a partial success rather than an unsupported feature.

## Expected

One of:

1. Compile the binding — emit the `K2Node_VariableGet` and wire it, which is exactly what the
   workaround below does by hand; or
2. Reject the argument with a typed error naming the pin; or
3. At minimum, return it in `warnings` so `warnings: []` stops meaning "everything you wrote landed".

Silence plus `nodesCreated: 3` currently means "three nodes exist", not "your graph was built".

## Workaround (works, 6 extra calls per graph)

Author the pose chain with AGIR, then wire the value pins through the generic graph verbs:

```
blueprint.graph.create_node {assetPath, graphName:"AnimGraph", nodeType:"VariableGet", variableName:"Direction", x,y}
blueprint.graph.connect_pins {assetPath, graphName:"AnimGraph", fromNodeId:<var>, fromPinName:"Direction", toNodeId:<blendspace>, toPinName:"X"}
```

`blueprint.graph.get_nodes` then reports `linkedTo` populated, and the Blueprint compiles clean.
The exposed pins already exist on the AGIR-created node, so nothing needs
`animation.authoring.set_anim_graph_pin_exposed` first.

## Wiki

`anim.md` § "AGIR text-form field-name conventions" documents the argument syntax for pose links and
node properties and says "live decompile output is the source of truth" — but a hand-authored graph
with real variable bindings decompiles *without* them (that is this bug), so the documented
round-trip cannot teach the correct form either. Whichever way this is fixed, the page should state
plainly whether AGIR can express a data-pin binding at all.

severity rationale: impact=silent false-success on a normal path (a graph reported as compiled is
missing its inputs, and the result animates convincingly enough to pass a glance) x reach=common
(every locomotion blend space and every aim offset in an authored Anim BP takes variable inputs)
-> High

## Fix

Fixed together with `B-decompile-agir-omits-wired-data-pin-bindings` — the same hole seen from the
write end. Expected option 1 ("compile the binding — emit the `K2Node_VariableGet` and wire it")
is what shipped; the workaround's 6 extra calls per graph are no longer needed.

`$Variable` and the new `bind <path>` form are recognised by `AGIRPinBindings::IsBindingValue` and
applied by `AGIRPinBindings::ApplyBindingArg`, reached through the single new write path
`AGIRCliff::Helpers::WriteAnimNodeArg`. `$Name` resolves the member on the target AnimBP, creates a
self-context `UK2Node_VariableGet`, and connects it to the pin (exposing a hidden pin first via
`SetPinVisibility`); a failed connection removes the getter rather than leaving a half-applied
binding behind. All seven arg-write sites now route through it, so a binding is honoured on any
anim-node family — `call`, `blend_space` (head and sample-graph bodies), `layered_blend`,
`linked_anim`, `linked_input_pose`, `save_cached_pose`, `use_cached_pose` — not just the generic
`call` arm.

Expected option 3 is also satisfied for the residual cases: an unresolvable `$Name` (variable
absent, pin absent, connection refused) returns a diagnostic that the existing arg loops surface as
an `AGIR_FIELD_WRITE` warning naming the variable and the pin, so `warnings: []` stops meaning
"three nodes exist".

Files changed: see the Fix section of `B-decompile-agir-omits-wired-data-pin-bindings` (same
change set). The compile-side files are `AGIRPinBindings.{h,cpp}` (new),
`AGIRCompilerHelpers.{h,cpp}`, `AGIRCompiler.cpp`, `AGIRCompiler_BlendSpace.cpp`,
`AGIRCompiler_CachedPose.cpp`, `AGIRCompiler_LayeredBlend.cpp`, `AGIRCompiler_LinkedAnim.cpp`,
`AGIRCompiler_LinkedInputPose.cpp`.

**Reviewer verification.** NOT compiled and NOT run here.

1. `PinWright.AGIR.PinBindings.VariableGetRoundTrip` covers the exact repro shape: compile AGIR
   carrying `$Var` on a data pin into a fresh AnimBP, then assert the pin is linked to a
   `UK2Node_VariableGet` reading that member — the assertion the old code fails.
   `PinWright.AGIR.PinBindings.PropertyBindingRoundTrip` covers the `bind` form.
2. Live: re-run this ticket's own repro on a scratch AnimBP that already declares `Direction`,
   `Speed` and `AimYaw`. `blueprint.graph.get_nodes {graphName:"AnimGraph"}` must now report
   `linkedTo` populated on the blend space's `X`/`Y` and the aim offset's `X`, and
   `blueprint.compile` must stay clean. Note the round-trip cross-check this ticket's repro used is
   now meaningful, because the sibling ticket made `anim.decompile_agir` print the bindings.
3. Negative case: compile `X: $NoSuchVariable` and confirm the response carries an
   `AGIR_FIELD_WRITE` warning naming `NoSuchVariable` and `X` instead of returning `warnings: []`.

## History
- `#1-filed` `OPEN` reporter — Hit while authoring `/Game/FPS/AI/ABP_Enemy` for the FPS AI stream: a
  rifle locomotion blend space plus a rifle aim offset plus a montage slot. The AGIR compile
  reported `nodesCreated:3, warnings:[]`; `anim.decompile_agir` and `blueprint.graph.get_nodes` both
  showed the `X`/`Y` value pins unwired. Recovered with three `blueprint.graph.create_node
  {nodeType:"VariableGet"}` + three `blueprint.graph.connect_pins` calls, after which
  `get_nodes` reports `Direction -> X`, `Speed -> Y` on the blend space and `AimPitch -> X` on the
  aim offset, and `blueprint.compile` is clean. Two adjacent facts for whoever fixes this: the pose
  links and the `BlendSpace:` asset argument in the *same* argument list compiled correctly, so the
  drop is specific to `$Variable` values; and no plugin source was read, the diagnosis is entirely
  from the compile response, the decompile, and the node readback.
- `#2-compiled-via-shared-binding-model` `IN-REVIEW` developer — Fixed alongside
  `B-decompile-agir-omits-wired-data-pin-bindings` via one shared `AGIRPinBindings` unit; `$Variable`
  now creates and wires the getter (Expected option 1) and an unresolvable one warns (option 3).
  Routed through a single `WriteAnimNodeArg` so all seven per-family arg loops honour it, not just
  the generic `call` arm. Not compiled and not run here — verification is the reviewer's.
