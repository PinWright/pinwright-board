---
id: B-decompile-agir-omits-wired-data-pin-bindings
title: "anim.decompile_agir omits data-pin bindings that ARE wired — a correct AnimGraph decompiles identically to a broken one, and it is the only AGIR read route because blueprint.decompile refuses anim graphs"
status: OPEN
severity: High
category: bug
tags: [anim, agir, decompile_agir, anim-blueprint, readback, blend-space, aim-offset, pin-binding, false-negative]
encounters: 1
lastSeen: 2026-09-02T21:35:00Z
---

# A working AnimGraph decompiles as if its inputs were never connected

`B-agir-variable-pin-args-silently-dropped` is the compile side: `anim.compile_agir`
accepts `X: $Variable` on an anim-node data pin and throws it away. This is the **read**
side of the same hole, and it is worse in one specific way — it makes a *correct* graph
unreadable. After that ticket's workaround (hand-wiring the pins with
`blueprint.graph.create_node {nodeType:"VariableGet"}` + `blueprint.graph.connect_pins`),
the graph is right and `anim.decompile_agir` still emits the same bare node lines. The two
states — "bindings dropped at compile" and "bindings present and working" — produce
**byte-identical AGIR**.

## Repro (read-only, on an asset whose pins are known-good)

`/Game/FPS/AI/ABP_Enemy` — the Anim BP from the ticket above, after its pins were repaired.

```
anim.decompile_agir {assetPath:"/Game/FPS/AI/ABP_Enemy"}
```

```
entry anim_graph AnimGraph {
    output %rotation_offset_blend_space_0
    %blend_space_player_0 = call `...AnimGraphNode_BlendSpacePlayer`(BlendSpace: "...MM_BS_Locomotion_2D")
    %slot_0 = call `...AnimGraphNode_Slot`(Source: %blend_space_player_0)
    %rotation_offset_blend_space_0 = call `...AnimGraphNode_RotationOffsetBlendSpace`(BasePose: %slot_0, BlendSpace: "...MM_AO_Rifle")
}
```
`warnings: []`.

The same graph through the generic reader:

```
blueprint.graph.get_nodes {assetPath:"/Game/FPS/AI/ABP_Enemy", graphName:"AnimGraph"}
```

```
AnimGraphNode_BlendSpacePlayer_0  X <- K2Node_VariableGet_0 "Get Direction"
                                 Y <- K2Node_VariableGet_1 "Get Speed"
AnimGraphNode_RotationOffsetBlendSpace_0  X <- K2Node_VariableGet_2 "Get AimPitch"
```

Three `K2Node_VariableGet` nodes exist in the graph and are linked. AGIR shows none of
them: not as arguments, not as separate statements, not as a warning.

## Why this is High and not cosmetic

- **It manufactures false defects.** Reviewing this Anim BP, the AGIR read is the
  documented route (`blueprint.decompile` on an Anim BP answers
  `"Skipped anim-graph AnimGraph — use AGIR (anim.decompile_agir) instead"`, so there is no
  second opinion inside the `blueprint.*` namespace). Reading it at face value says the
  locomotion blend space and the aim offset take no input — i.e. the character is frozen at
  the blend space origin and never aims. That is a specific, serious, and **wrong**
  conclusion about shipped content, and it is the conclusion the tool hands you. It took a
  `blueprint.graph.get_nodes` cross-check to retract it.
- **It also hides the real bug it should expose.** Because the two states are
  indistinguishable, AGIR cannot be used to verify the fix for
  `B-agir-variable-pin-args-silently-dropped` — the very check that ticket's own repro
  section performs ("immediately decompiling the same asset shows ... every `$Variable`
  argument gone") proves nothing, since it prints that either way.
- **`warnings: []`.** The decompiler has warning machinery (`blueprint.decompile` emits
  orphan-node warnings on the same asset's EventGraph). Dropping a wired input pin without
  one is a silent lossy read.

## Expected

`anim.decompile_agir` must represent a connected data pin. In descending order of
preference:

1. Emit it in the argument list as `X: $Direction` — the same spelling
   `anim.compile_agir` accepts (and should honour, per the sibling ticket), so the
   round-trip closes.
2. Emit it as an explicit node + link, whatever spelling AGIR settles on for a
   `K2Node_VariableGet` inside an anim graph.
3. At minimum, `warnings: [{text:"AnimGraph node <n> pin X is connected to <node>; the
   binding is not represented in AGIR", severity:"warn"}]`, so a reader knows the text is
   incomplete rather than complete-and-empty.

Whichever lands, `anim.md` should stop saying "live decompile output is the source of
truth" until the decompile actually is one.

## Workaround

Never read an AnimGraph's data-pin wiring from AGIR. Use
`blueprint.graph.get_nodes {graphName:"AnimGraph"}` and read `pins[].linkedTo`; it reports
anim-graph nodes and their `K2Node_VariableGet` sources correctly.

severity rationale: impact=silent lossy read that produces a confident wrong answer about
working content, on the only documented read route for the asset kind x reach=every Anim
Blueprint with a variable-driven blend space or aim offset, i.e. effectively all of them
-> High

## History
- `#1-filed` `OPEN` reporter — Found while reviewing the FPS AI stream's `ABP_Enemy` as a
  critic, on UE 5.8 / `EAContentExamples58`. Read the graph through
  `anim.decompile_agir`, saw `BlendSpacePlayer(BlendSpace: ...)` with no `X`/`Y` and the
  aim offset with no `X`, and had written up "the enemy's locomotion blend space is pinned
  at the idle sample, so enemies slide with no walk cycle" as a top defect before
  cross-checking with `blueprint.graph.get_nodes` — which showed `Direction -> X`,
  `Speed -> Y` and `AimPitch -> X` all linked, and `ABP_Enemy`'s
  `BlueprintUpdateAnimation` computing all three. The defect was retracted. Related but
  distinct from `B-agir-variable-pin-args-silently-dropped` (that one is
  `anim.compile_agir` discarding the argument; this one is `anim.decompile_agir` not
  printing a binding that exists), and the two together mean AGIR can neither write nor
  read a data-pin binding while reporting success both ways. No plugin source read.
