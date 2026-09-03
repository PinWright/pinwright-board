---
id: E-cue-graph-verbs-cannot-set-firstnode
title: "No SoundCue verb can set FirstNode, so the documented add_cue_node + connect_cue_nodes workflow always produces a silently mute, rootless cue"
status: OPEN
severity: High
category: enhancement
tags: [audio, sound-cue, add-cue-node, connect-cue-nodes, firstnode, cue-root, silent-noop, decompile-sound-cue]
---

# The cue graph verbs build a correct tree and then cannot root it

`create_sound_cue` documents the intended workflow verbatim: *"For complex node graphs,
follow with audio.authoring.add_cue_node and connect_cue_nodes."* Following it exactly
produces a cue that **cannot play**. `add_cue_node` creates every node detached, and
`connect_cue_nodes` only ever writes `SourceNode->ChildNodes[ChildIndex]` — it wires nodes
*to each other*. Nothing in the namespace writes `USoundCue::FirstNode`, which is what the
audio device actually starts playback from, so however complete the tree is, the cue is
mute. `decompile_sound_cue` shows the state plainly (`orphan`, plus a `SoundCue has no
FirstNode.` warning), but that is a reader; there is no writer to act on it.

There is also no root pseudo-node to connect to: `connect_cue_nodes` with
`sourceNodeId: "Output"` returns `[SOURCE_NODE_NOT_FOUND] Source node not found: Output`.

The one path that does set `FirstNode` is `create_sound_cue`'s own pre-wire, and it is
reachable only at create time and only for the shapes it hardcodes (wave player, optional
looping, optional volume/pitch modulator). Any other root — a Random, a Mixer, a
Concatenator, a Modulator over a Random — is unreachable through typed verbs.

## Verbatim repro (this is the whole failure, in five calls)

```
audio.authoring.create_sound_cue   {"name":"SC_Impact_Concrete","path":"/Game/FPS/Audio/Cues"}
audio.authoring.add_cue_node       {"assetPath":"/Game/FPS/Audio/Cues/SC_Impact_Concrete","nodeType":"modulator"}   -> SoundNodeModulator_0
audio.authoring.add_cue_node       {"assetPath":"...","nodeType":"random"}                                          -> SoundNodeRandom_0
audio.authoring.add_cue_node       {"assetPath":"...","nodeType":"wave_player","wavePath":".../SW_Impact_Concrete_A"} -> SoundNodeWavePlayer_0
audio.authoring.add_cue_node       {"assetPath":"...","nodeType":"wave_player","wavePath":".../SW_Impact_Concrete_B"} -> SoundNodeWavePlayer_1
audio.authoring.connect_cue_nodes  {"assetPath":"...","sourceNodeId":"SoundNodeRandom_0","targetNodeId":"SoundNodeWavePlayer_0","childIndex":0}
audio.authoring.connect_cue_nodes  {"assetPath":"...","sourceNodeId":"SoundNodeRandom_0","targetNodeId":"SoundNodeWavePlayer_1","childIndex":1}
audio.authoring.connect_cue_nodes  {"assetPath":"...","sourceNodeId":"SoundNodeModulator_0","targetNodeId":"SoundNodeRandom_0","childIndex":0}
```

Every call returns a clean success. `decompile_sound_cue` on the result:

```
sound_cue `/Game/FPS/Audio/Cues/SC_Impact_Concrete.SC_Impact_Concrete` {
  volume 0.75
  pitch 1.0
  orphan modulator SoundNodeModulator_0 @(0, 0) (PitchMin: 1.0, PitchMax: 1.0, VolumeMin: 1.0, VolumeMax: 1.0) {
    child random SoundNodeRandom_0 @(0, 0) () {
      child wave_player SoundNodeWavePlayer_0 @(0, 0) (wave: `.../SW_Impact_Concrete_A...`)
      child wave_player SoundNodeWavePlayer_1 @(0, 0) (wave: `.../SW_Impact_Concrete_B...`)
    }
  }
}
warnings: ["SoundCue has no FirstNode."]
```

The tree is exactly right and the cue is silent. Compare `create_sound_cue` with a
`wavePath`, which yields `root wave_player SoundNodeWavePlayer_0 ...` and no warning — the
`orphan` / `root` distinction in SCIR is the whole defect, visible in one word.

## Attempted and rejected

`connect_cue_nodes {"sourceNodeId":"Output", ...}` -> `[SOURCE_NODE_NOT_FOUND] Source node
not found: Output`. No `set_cue_root` / `set_cue_first_node` verb exists in
`audio.authoring`.

## Workaround (used to ship 14 cues)

Two calls per cue, and the second is non-obvious:

1. `property.set {"objectPath":"/Game/FPS/Audio/Cues/SC_Impact_Concrete.SC_Impact_Concrete",
   "propertyName":"FirstNode",
   "value":"/Game/FPS/Audio/Cues/SC_Impact_Concrete.SC_Impact_Concrete:SoundNodeModulator_0"}`
   -> `applied: true`. (The node's object path uses a `:` subobject separator; discover it
   with `unreal.find_object(cue, "SoundNodeModulator_0")` — nothing in the RPC results
   prints it, they return only the bare `nodeId`.) `USoundCue::AllNodes` is `protected` and
   unreadable from Python, so there is no scripted way to enumerate node paths either.
2. **Then re-issue one `connect_cue_nodes` that is already satisfied**, purely for its side
   effect: the handler calls `Cue->LinkGraphNodesFromSoundNodes()`, and that engine routine
   is what links the EdGraph Root node's pin to `FirstNode`. Without this, `FirstNode` is
   correct in the serialised asset but the Sound Cue editor's Output node shows unconnected,
   and the next graph edit a human makes recompiles `FirstNode` back to null from the
   unlinked graph. A caller has no way to know this from the docs.

## Suggested fix

Add `audio.authoring.set_cue_root {assetPath, nodeId, save}` that assigns `FirstNode` and
calls `LinkGraphNodesFromSoundNodes()`, so both halves of the workaround collapse into one
documented verb. Alternatively let `connect_cue_nodes` accept a reserved
`sourceNodeId: "Output"` (or `"Root"`) meaning "make this the cue root" — the error message
already suggests callers try it. Either way `decompile_sound_cue`'s existing
`SoundCue has no FirstNode.` warning should name the verb that fixes it, and
`create_sound_cue`'s "follow with add_cue_node and connect_cue_nodes" sentence should not
describe a workflow that cannot finish.

## History
- `#1-filed` `OPEN` reporter — Hit while building 14 impact/footstep random-selector cues under `/Game/FPS/Audio/Cues/`. Followed `create_sound_cue`'s own documented follow-on workflow exactly; all eight calls returned clean success and `decompile_sound_cue` showed a perfectly-shaped `modulator -> random -> 2x wave_player` tree marked `orphan`, with `SoundCue has no FirstNode.` in `warnings` — a cue that would never make a sound. `connect_cue_nodes` with `sourceNodeId: "Output"` was rejected `SOURCE_NODE_NOT_FOUND`, and no verb in `audio.authoring` writes `FirstNode`. Shipped by falling back to `property.set` on `FirstNode` (node object paths use a `:` subobject separator and are not returned by any RPC — `USoundCue::AllNodes` is `protected` and unreadable from Python, so I had to find them with `unreal.find_object`), followed by a redundant `connect_cue_nodes` re-issued solely to trigger the handler's `LinkGraphNodesFromSoundNodes()` and link the EdGraph Root pin. Both steps are undiscoverable from the wiki. Related but distinct: `B-create-sound-cue-looping-noop` covers `create_sound_cue` dropping `looping`/`volume`/`pitch` when `wavePath` is omitted and mentions the same `property.set` workaround in passing; this ticket is the missing capability in the graph-editing verbs themselves, which that ticket's fix does not address. Severity High: impact = the documented complex-graph workflow silently produces an unplayable asset x reach = every SoundCue authored with more than the create-time pre-wire.
