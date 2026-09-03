---
id: B-decompile-agir-omits-wired-data-pin-bindings
title: "anim.decompile_agir omits data-pin bindings that ARE wired — a correct AnimGraph decompiles identically to a broken one, and it is the only AGIR read route because blueprint.decompile refuses anim graphs"
status: IN-REVIEW
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

## Fix

Both binding forms now round-trip, in one shared unit rather than two per-side patches.

**New:** `Source/PinWright/Private/AGIR/AGIRPinBindings.{h,cpp}` — the AGIR vocabulary for
anim-node *data*-pin bindings, used by the emitter (read) and by every compile-side arg loop
(write):

- `X: $Direction` — a `UK2Node_VariableGet` for that AnimBP member wired into the pin. Same
  `$Name` sigil BPIR already uses for a member read, so one concept has one spelling across the
  IR family.
- `Alpha: bind Speed` / `Alpha: bind Struct.Member` / `Alpha: bind fn GetAlpha` — a
  property-access binding (`FAnimGraphNodePropertyBinding`).

The `PropertyBindings` map is reached reflectively (`Binding` object on UE 5.4+, the node's own
map on 5.3) so no engine `Internal/`/`Private/` header is needed and both layouts work with one
code path. The compile side writes `PropertyName` / `PropertyPath` / `PathAsText` / `Type` /
`bIsBound` / `ArrayIndex` and calls `ReconstructNode()`; the engine's `OnReconstructNode`
recomputes the derived `PinType` / `PromotedPinType` / `bIsPromotion`. A hidden pin is exposed
via `SetPinVisibility` before binding.

**A bound pin's literal is suppressed from the reflected pass** (`AppendReflectedNodeFields` now
takes a suppressed-name set). That is what makes "driven, currently 0" textually different from
"not driven" — the distinction encounter #2 opened the file to make.

**Unrepresentable links now warn instead of vanishing.** `FAGIRTextEmitter` grew a `Warnings`
array; a wired pin whose driver is not a plain self-context member getter produces
`AGIR_PIN_BINDING_NOT_REPRESENTABLE: node '<n>' pin '<p>' is driven by '<title>' ...`, which
`FAGIRDecompiler` appends to its result and `anim.decompile_agir` relays.

Files changed:
- `Source/PinWright/Private/AGIR/AGIRPinBindings.h` (new)
- `Source/PinWright/Private/AGIR/AGIRPinBindings.cpp` (new)
- `Source/PinWright/Private/AGIR/AGIRTextEmitter.{h,cpp}` — `AppendNodeFields` collapses the
  pose-links + bindings + reflected-fields sequence at all 7 emit sites; `Warnings` sink
- `Source/PinWright/Private/AGIR/AGIRDecompiler.cpp` — relays emitter warnings
- `Source/PinWright/Private/AGIR/AGIRCompilerHelpers.{h,cpp}` — `WriteAnimNodeArg`, the single
  write path (binding value -> binding, else the reflective struct write)
- `AGIRCompiler.cpp`, `AGIRCompiler_BlendSpace.cpp`, `AGIRCompiler_CachedPose.cpp`,
  `AGIRCompiler_LayeredBlend.cpp`, `AGIRCompiler_LinkedAnim.cpp`,
  `AGIRCompiler_LinkedInputPose.cpp` — 7 arg-write sites routed through it
- `Source/PinWright/Private/Tests/Assets/TestAGIRPinBindings.cpp` (new) — 2 round-trip tests
- `docs/wiki-src/anim.md` — new "AGIR data-pin bindings" section; "Historical, now fixed" entry

**Reviewer verification.** NOT compiled and NOT run by this change — a separate compile pass
follows. Then:

1. Run `PinWright.AGIR.PinBindings.VariableGetRoundTrip` and
   `PinWright.AGIR.PinBindings.PropertyBindingRoundTrip`. The first hand-wires a getter into a
   real AnimGraph, asserts the decompile prints `Alpha: $PinWrightBoundAlpha` *and* not the pin's
   dead `0.25` literal, then compiles into a fresh AnimBP and asserts the link was rebuilt to the
   same member. The second compiles hand-authored `Alpha: bind <var>` text, asserts the engine's
   public `HasBinding("Alpha")`, and re-decompiles to the same spelling.
2. Live check on the reported asset: `anim.decompile_agir {assetPath:"/Game/FPS/AI/ABP_Enemy"}`
   must now show `X: $Direction`, `Y: $Speed`, `X: $AimPitch` and `Alpha: $CrouchAlpha` on the
   `ModifyBone` and both `TwoBoneIK` nodes — cross-check against
   `blueprint.graph.get_nodes {graphName:"AnimGraph"}`, which reported those six links.
3. Recompile that decompile output into a scratch AnimBP and diff a second decompile against the
   first; the bindings must survive.

Counterfactuals: removing `AppendBindingFields` from `AppendNodeFields` makes the text assertions
fail; removing the `$`/`bind` branch from `WriteAnimNodeArg` makes the rebuilt-link and
`HasBinding` assertions fail.

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
- `#2-reproduces-on-rebuilt-graph-and-now-hides-a-disabled-alpha` `OPEN` reporter — Still reproduces
  at HEAD on the same asset after the AI stream rebuilt its AnimGraph (`/Game/FPS/AI/ABP_Enemy`,
  UE 5.8 / `EAContentExamples58`), and the consequence has got worse because the graph now carries
  skeletal controls whose alphas are the thing under review. `anim.decompile_agir` prints:
  `BlendSpacePlayer(BlendSpace: "...MM_BS_Locomotion_2D")`, `RotationOffsetBlendSpace(BasePose: ...,
  BlendSpace: "...MM_AO_Rifle")`, `ModifyBone(ComponentPose: ..., BoneToModify: "(BoneName=\"pelvis\")",
  Translation: "(X=0,Y=0,Z=-34)", TranslationMode: "BMM_Additive")` and two `TwoBoneIK(...)` — every
  one of them with **no `Alpha:` argument and no `X:`/`Y:` argument**. `blueprint.graph.get_nodes
  {graphName:"AnimGraph"}` on the same asset in the same minute reports six live links the AGIR does
  not mention: `Get Direction -> BlendSpacePlayer.X`, `Get Speed -> BlendSpacePlayer.Y`,
  `Get AimPitch -> RotationOffsetBlendSpace.X`, and one `Get CrouchAlpha` fanning out to
  `ModifyBone.Alpha`, `TwoBoneIK(foot_l).Alpha` and `TwoBoneIK(foot_r).Alpha`.
  **Why this is now a correctness trap rather than only a lossy read:** a skeletal-control node with
  an unbound `Alpha` sits at its literal default and a bound one is driven per frame, and this
  project's build had *deliberately* pinned `CrouchAlpha` to `0.0` in `BlueprintUpdateAnimation` to
  disable a broken crouch pose. So the AGIR text for the live graph is character-for-character what
  AGIR would print for a graph whose alphas were never wired at all — the decompile cannot
  distinguish "driven, currently 0" from "not driven", which is exactly the distinction a reviewer
  opens the file to make. I only established the real state by reading
  `blueprint.decompile` of the event graph and `blueprint.graph.get_nodes` of the anim graph
  side by side. The `Expected` section above is unchanged and still the right fix; add the
  skeletal-control `Alpha` pin to the set of bindings that must round-trip. No plugin source read.
- `#3-fixed-both-binding-forms` `IN-REVIEW` developer — Implemented per the Fix section above:
  `$Name` for a graph-wired variable getter and `bind <path>` for a property-access binding, emitted
  by the decompiler and honoured by the compiler through one shared `AGIRPinBindings` unit, with the
  bound pin's dead literal suppressed and unrepresentable drivers reported as
  `AGIR_PIN_BINDING_NOT_REPRESENTABLE` warnings. Two round-trip tests added. Not compiled and not run
  here — verification is the reviewer's, per the Fix section.
