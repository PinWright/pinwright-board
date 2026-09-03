---
id: F-metasound-array-inputs-cannot-be-populated
title: "Array-typed MetaSound inputs and pins can be CREATED but never POPULATED, so the whole Array.* node family (Random Get, Shuffle, Get, Set, Concat) is unreachable from the RPC surface"
status: OPEN
severity: High
category: feature
tags: [audio, metasound, array, set_metasound_default, set_metasound_node_input_default, random, round-robin, unreachable-node-family]
encounters: 1
lastSeen: 2026-09-03T01:10:00Z
---

# Array-typed MetaSound state can be declared but not filled

`audio.authoring.add_metasound_input` happily accepts an array data type —
`{inputName: "ZZ_ArrayProbe", inputType: "WaveAsset:Array"}` returns
`{"inputName":"ZZ_ArrayProbe","inputType":"WaveAsset:Array","nodeId":"..."}` and the input really is
created. But nothing can then put a value in it:

```
audio.authoring.set_metasound_default {inputName:"ZZ_ArrayProbe",
    objectValue:"/Game/FPS/Audio/Waves/Weapons/SW_Fire_AR_Mech"}
-> [INVALID_ASSET_TYPE] MetaSound data type 'WaveAsset:Array' accepts no object literal, so
   objectValue '...' cannot be bound to it; nothing was written.
   Use floatValue / intValue / boolValue / stringValue for this type.
```

The four alternatives the error names are all **scalars**, so none of them can express an array of
object references. `set_metasound_node_input_default` has the same five value parameters
(`floatValue`, `intValue`, `boolValue`, `stringValue`, `objectValue`) and therefore the same ceiling
on a node's `In Array` pin.

## Why this makes a whole node family dead

The registry ships a full array suite for every data type — for `WaveAsset` alone:
`Array.Random Get`, `Array.Shuffle`, `Array.Get`, `Array.Set`, `Array.Concat`, `Array.Subset`,
`Array.Num`, `Array.GetLastIndex`. **Every one of them takes an array as input**, and there is no
"make array" node: an array enters a graph only as a literal on a graph input or a node pin. With
literals unreachable, the entire family is unreachable. `Array.Set` looks like an escape hatch
(`Trigger, Array, Index, Value -> Array`) but it still needs a seed array to write into.

The concrete cost, from the FPS AUDIO stream: `Array.Random Get (WaveAsset:Array)` is the idiomatic
UE way to round-robin a weapon layer over several samples — it even has a `No Repeats` input for
exactly this. It cannot be used. The fallback is arithmetic selection built from
`UE.RandomFloat` -> `UE.Multiply.Float` -> `Clamp.Clamp.Float` -> `UE.Subtract.Float` driving a
per-variant `UE.Multiply.Audio by Float`, which costs about 4 nodes for a fair 2-way switch and
about 16 for a fair 3-way, against **one node** for the array route. That cost difference is what
forced this build down from three variants per layer to two.

**Workaround:** build the selector from scalar math nodes as above, and keep the variant count low
enough that the node explosion stays manageable. There is no workaround that reaches the `Array.*`
nodes themselves.

**Fix:** accept an array form on the two default-setting verbs — an `arrayValue` parameter taking a
JSON array whose element form matches the declared element type (`objectValue` semantics per entry
for `*:Array` object types, numbers for `Float:Array`/`Int32:Array`, and so on). `Float:Array` is
worth calling out separately: `Array.Random Get`'s `Weights` pin is a `Float:Array`, so even the
weighting of an array node is unreachable today, which is a near-exact echo of
`B-cue-random-node-weights-zero-always-picks-first` on the SoundCue side — an authoring path that
leaves a random node's weights unpopulated and therefore inert.

## History
- `#1-filed` `OPEN` reporter — Found while adding per-shot round-robin to the weapon layers of the FPS build on EAContentExamples58 (UE 5.8), the standing rubric-row-6 defect the critic has scored low for three review rounds. `add_metasound_input` accepted `inputType: "WaveAsset:Array"` and created the input; `set_metasound_default` then refused every way of filling it, answering `INVALID_ASSET_TYPE ... accepts no object literal ... Use floatValue / intValue / boolValue / stringValue`, none of which can carry an array of object references. `set_metasound_node_input_default` exposes the same five scalar value parameters, so a node's `In Array` pin is equally unreachable. Since no "make array" node exists and every `Array.*` node consumes an array it cannot be handed, the whole family is dead from the RPC surface — including `Array.Random Get (WaveAsset:Array)`, which has a `No Repeats` input designed precisely for the round-robin being built. Verified the probe input was really created and then removed it cleanly with `remove_metasound_input`, so nothing was left behind. Workaround adopted: an arithmetic selector (`UE.RandomFloat` -> `UE.Multiply.Float` -> `Clamp.Clamp.Float` -> `UE.Subtract.Float` -> `UE.Multiply.Audio by Float` per variant), roughly 4 nodes for a fair 2-way switch versus 1 node for the array route, and about 16 for a 3-way — which is why this build ships two variants per layer rather than three. Asks for an `arrayValue` parameter on both default-setting verbs; note that `Array.Random Get`'s `Weights` pin is a `Float:Array` and so is unreachable too, making this the MetaSound-side twin of `B-cue-random-node-weights-zero-always-picks-first`.
