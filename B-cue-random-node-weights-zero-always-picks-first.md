---
id: B-cue-random-node-weights-zero-always-picks-first
title: "SoundCue authoring leaves USoundNodeRandom::Weights zero-filled, so the Random node is deterministic and always plays child 0 — every readback shows a correct graph"
status: OPEN
severity: High
category: bug
tags: [audio, audio.authoring, sound-cue, add_cue_node, connect_cue_nodes, sound-node-random, weights, silent-no-op, randomization, decompile_sound_cue, describe_sound_cue]
encounters: 2
lastSeen: 2026-09-03T00:40:00Z
---

# A cue built through the RPC surface has a Random node that never randomises

Building a variation cue the documented way — `create_sound_cue`, then `add_cue_node`
(`SoundNodeRandom`), then `add_cue_node` per `SoundNodeWavePlayer`, then `connect_cue_nodes` to
parent the wave players under the random node — produces a cue whose `USoundNodeRandom::Weights`
array is **`[0.0, 0.0]`**. The engine's selection then always returns child 0.

This is a silent no-op of the worst shape: **every readback the surface offers reports a correct
graph.** `decompile_sound_cue` prints the random node with both children; `describe_sound_cue`
lists `ChildNodes` correctly and shows `firstNode` set; the cue opens in the editor looking right.
The `Weights` array is printed — `(Weights: [0.000000,0.000000])` — but nothing says that value is
degenerate, and a caller has no reason to read a weight array as the on/off switch for the node's
entire purpose.

## Why zero weights disable the node (engine source, UE 5.8)

`USoundNodeRandom::ChooseNodeIndex`
(`Engine/Source/Runtime/Engine/Private/SoundNodeRandom.cpp:106-162`):

```cpp
int32 NodeIndex = 0;
float WeightSum = 0.0f;
for (int32 i = 0; i < Weights.Num(); ++i) { ... WeightSum += Weights[i]; }   // -> 0.0
float Choice = FMath::FRand() * WeightSum;                                   // -> 0.0
WeightSum = 0.0f;
for (int32 i = 0; i < ChildNodes.Num() && i < Weights.Num(); ++i) {
    ...
    WeightSum += Weights[i];              // stays 0.0
    if (Choice < WeightSum) { ... break; } // 0.0 < 0.0 is false, never taken
}
return NodeIndex;                          // still the initialiser: 0
```

The loop can never break, so `NodeIndex` keeps its initial `0` and **child 0 plays every time**.

`FixWeightsArray` (`:39-51`) is not a safety net — it only resizes, and it grows the array with
`Weights.AddZeroed(...)`, which is exactly where the zeros come from. It runs from `PostLoad` and
`PostEditImport`, so a reload does not repair the cue either.

The engine's own intended default is 1.0, set on the editor's insert path
(`USoundNodeRandom::InsertChildNode`, `:282-291`):

```cpp
Weights.InsertUninitialized( Index );
Weights[Index] = 1.0f;
```

So a cue authored by hand in the editor works and a cue authored through the RPC surface does not,
because the RPC path parents children without going through `InsertChildNode`.

## Evidence from real content

`/Game/FPS/Audio/Cues/` — 13 variation cues built in the AUDIO stream's Build 02 specifically to
fix "every footstep and every bullet impact is bit-identical" (`Docs/fps/reviews/audio-review-01.md`
defect 3). Every one has the same shape and the same dead weights:

```
call("audio.authoring.decompile_sound_cue", {assetPath:"/Game/FPS/Audio/Cues/SC_Impact_Concrete"})
-> root modulator SoundNodeModulator_0 (PitchMin: 0.917, PitchMax: 1.091, VolumeMin: 0.8, VolumeMax: 1.0) {
     child random SoundNodeRandom_0 (Weights: [0.000000,0.000000]) {
       child wave_player SoundNodeWavePlayer_0 (wave: .../SW_Impact_Concrete_A)
       child wave_player SoundNodeWavePlayer_1 (wave: .../SW_Impact_Concrete_B)
     }
   }
```

`describe_sound_cue` on `SC_Step_Metal` likewise reports `firstNode` set, both children parented,
and `"Weights": {"value":[0,0]}`. The `_B` variants are referenced, loaded and cooked, and are
unreachable at runtime. The build's fix measures as done by every check available and changes
nothing a player hears — the defect it was meant to close is still fully present.

## What it should do

`add_cue_node` / `connect_cue_nodes` should reproduce the editor's contract when the parent is a
`USoundNodeRandom` (and any other node type whose per-child arrays carry meaning): route the child
attachment through `InsertChildNode`, or explicitly set `Weights[NewIndex] = 1.0f` and grow
`HasBeenUsed` to match, so a node created through the RPC behaves like one created in the editor.

Two smaller mitigations worth having regardless:

- **`validate_metasound` has no cue analogue.** A `validate_sound_cue` (or a finding in
  `audio.analysis.audit_folder`) that flags a `SoundNodeRandom` whose `Weights` sum to zero, or a
  cue with a null `FirstNode`, would catch this class without the caller knowing the engine
  internals. A zero weight sum has no legitimate use: it is always a node that cannot choose.
- **`decompile_sound_cue` should warn.** It already prints the array and already has a `warnings`
  channel that came back empty here; `Weights sum to 0 — this node always selects child 0` costs
  one line and turns a silent defect into a visible one.

Related but distinct: `E-cue-graph-verbs-cannot-set-firstnode` (the root pointer, a different
per-cue field the authoring path also does not populate the way the editor does). Same root shape —
the RPC surface builds cue graphs without the editor-side bookkeeping that makes them function —
so a fix should probably audit every `USoundNode` subclass with parallel per-child state.

severity rationale: impact=silent no-op that defeats the feature the node exists for, with every
available readback reporting success x reach=any variation cue built through the surface, which is
the standard way to author footstep/impact/weapon variation -> High

**Workaround:** after parenting children, set the weights explicitly (e.g. via `property.set` on
the node object, `Weights` = `[1.0, 1.0]`), then re-read with `decompile_sound_cue` and confirm the
printed array is non-zero before believing the cue randomises.

## History
- `#1-filed` `OPEN` reporter — Found while reviewing the AUDIO stream's Build 02
  (`Docs/fps/reviews/audio-review-02.md`) against review 01's defect 3, "every impact and footstep
  is bit-identical". The build replaced 16 raw-SoundWave entries in `/Game/FPS/Audio/DA_ImpactSFX`
  with 14 SoundCues under `/Game/FPS/Audio/Cues/`, each `Modulator -> Random -> [wave_A, wave_B]`,
  and the build report verified them by presence, by `decompile_sound_cue` shape, and by counting
  `SoundNodeRandom` occurrences — all of which pass. `decompile_sound_cue` on `SC_Impact_Concrete`
  and `describe_sound_cue` on `SC_Step_Metal` both show `Weights: [0.0, 0.0]`, and
  `USoundNodeRandom::ChooseNodeIndex` (SoundNodeRandom.cpp:106-162) cannot break out of its
  selection loop when the weight sum is zero, so `NodeIndex` keeps its `0` initialiser and child 0
  plays on every trigger. `FixWeightsArray` (:39-51) only resizes and grows with `AddZeroed`, so
  PostLoad does not repair it; the editor's `InsertChildNode` (:282-291) sets `Weights[Index] = 1.0f`,
  which is the contract the RPC path misses. Net effect: 13 cues that measure as correct always play
  their `_A` variant, and the review's most audible defect survives a fix that every check called
  done. Asks for the insert path to set 1.0 (or call `InsertChildNode`), plus a zero-weight-sum
  warning in `decompile_sound_cue` and/or an `audit_folder` finding.
- `#2-confirmed-by-owning-stream-and-repaired` `OPEN` reporter — Confirmed by the AUDIO stream that authored the cues, and repaired in content. Reading every cue's node tree from `first_node` (the `AllNodes` UPROPERTY is **not exposed to Python** — `get_editor_property('all_nodes')` raises `Failed to find property 'all_nodes'`, so the tree must be walked through `child_nodes`), **all 13 `SoundNodeRandom` nodes across the 14 cues held `Weights: [0.0, 0.0]`** — 100 %, not a subset. The fourteenth cue, `SC_Impact_Generic`, has no Random node by design. That rules out a partial or race-dependent cause: the authoring path produced zero weights on every node it created, which matches the ticket's `AddZeroed` diagnosis. Repair is a single per-node property write — `set_editor_property('weights', [1.0] * len(child_nodes))` — followed by `EditorLoadingAndSavingUtils.save_packages`. Verified two independent ways rather than by read-back: `decompile_sound_cue` now prints `child random SoundNodeRandom_0 @(0, 0) (Weights: [1.000000,1.000000])`, and a byte scan of the saved `.uasset`s finds the little-endian `1.0,1.0` float pair in **13 of 13** cues that have a Random node. Worth adding to the ticket's ask: the defect is invisible to every readback the project had been using — `describe_sound_cue` reports the node and its children, `decompile_sound_cue`'s `warnings` array is empty, and `audit_folder` has no finding for it — so a zero weight-sum warning in `decompile_sound_cue` would have caught this at authoring time. Note also that `decompile_sound_cue` *does* print the weights array, so the information was on screen in Build 02 and simply was not checked; the critic reading the same output is what found it.
- `#3-repair-verified-independently-warning-still-absent` `OPEN` reporter — Third-round review verification, recorded because the repair is content-side and the plugin defect is untouched. All 14 cues under `/Game/FPS/Audio/Cues/` re-checked two independent ways: a byte scan of each saved `.uasset` for the little-endian `1.0f,1.0f` pair finds exactly one in each of the 13 cues that own a `SoundNodeRandom` and none in `SC_Impact_Generic` (which has no Random node by design), and `decompile_sound_cue` on all 14 now prints `(Weights: [1.000000,1.000000])` on every Random node. The content is fixed. **The plugin behaviour that produced the zeros is not**, and neither is the detection ask: `decompile_sound_cue` still returns `"warnings": []` on all 14 calls, so a freshly authored cue with a dead Random node would still read as correct today, and `audio.analysis.audit_folder` — which I ran over all 63 waves in the same session and which reports seven finding types — still has no cue-graph check of any kind. Worth noting for whoever fixes this: the repair had to be done with a per-node `set_editor_property('weights', ...)` walk, and the walk itself is awkward because `AllNodes` is not exposed to Python (`get_editor_property('all_nodes')` raises `Failed to find property 'all_nodes'`), so the tree must be traversed through `first_node` / `child_nodes`. A caller who does not already know the weights are wrong has no signal pointing them at that walk. The one-line warning asked for in `#1` remains the cheapest fix for the detection half, independent of whether the insert path is corrected.
