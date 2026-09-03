---
id: F-metasound-array-inputs-cannot-be-populated
title: "Array-typed MetaSound inputs and pins can be CREATED but never POPULATED, so the whole Array.* node family (Random Get, Shuffle, Get, Set, Concat) is unreachable from the RPC surface"
status: IN-REVIEW
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

## Fix

Verified TRUE, and the root cause is sharper than "no array param existed". `RegisterDataType`
registers `TArray<T>` through `RegisterDataTypeArrayWithFrontend`, which calls
`RegisterDataTypeWithFrontendInternal<TArrayType, ...>` **without** the `UClassToUse` template
argument (`MetasoundDataTypeRegistrationMacro.h`), so an array type's
`FDataTypeRegistryInfo::ProxyGeneratorClass` is null and only `bIsProxyArrayParsable` is set.
`IDataTypeRegistry::IsValidUObjectForDataType` tests `bIsProxyParsable`
(`MetasoundFrontendDataTypeRegistry.cpp:1059`), so it rejects every object for `WaveAsset:Array`
and `GetUClassForDataType` returns null — which is exactly why the error read "accepts no object
literal ... use floatValue / intValue / boolValue / stringValue". The registry also exposes no
array→element relation, so element class validation has to go through the element type NAME.

**Design: one `arrayValue` param through the existing literal helper, not a parallel path.**
- `MakeArrayLiteralForMetaSoundType` (`MetaSoundLiteralFromTypeName.h/.cpp`) builds an array
  `FMetasoundFrontendLiteral` from a JSON array. The element form is taken from the ARRAY type's
  own registry literal shape (`GetDesiredLiteralType` → FloatArray / IntegerArray / BooleanArray /
  StringArray / UObjectArray / NoneArray), never guessed from the JSON — the same direction the
  scalar path already takes, so a `Float:Array` refuses `"0.25"` and an `Int32:Array` refuses
  `1.7` instead of coercing. Object entries are resolved and class-checked by calling the
  EXISTING `MakeObjectLiteralForMetaSoundType` against the ELEMENT type, so the
  `WaveAsset` → `USoundWave` pairing still comes from the registry and is nowhere hardcoded. An
  empty array is a legal value (it clears the default).
- `BuildMetaSoundLiteralFromParams` (`MetaSoundLiteralParams.h`) — the one helper all four value
  verbs already share — gained the branch, so `set_metasound_default`,
  `set_metasound_node_input_default`, `add_metasound_variable` and
  `set_metasound_variable_default` all get `arrayValue` from a single edit. Array and scalar are
  exclusive and the TARGET's declared type decides which applies: an array-typed target refuses
  every scalar param naming `arrayValue` as the way out, and a scalar-typed target refuses
  `arrayValue` rather than ignoring it.
- `CanonicalizeMetaSoundTypeName` now canonicalizes through the element (`Int:Array` →
  `Int32:Array`), so the documented convenience aliases no longer work for a scalar and silently
  fail for its array. New `IsMetaSoundArrayDataType` (registry `bIsArrayType`, falling back to the
  engine's `:Array` name shape) and `GetMetaSoundArrayElementTypeName`.
- `DescribeMetaSoundLiteral` emits `arrayNum` and, for object arrays, `objectPaths`, so the
  existing document read-back can be verified element by element instead of against one flattened
  `ToString()`.

Rejected alternative: a `Array.Make`-style synthesizing verb. There is no such node in the
registry, so it would have meant inventing a construct the engine does not have, and it would
still not reach a node pin's literal — which is the state `Array.Random Get`'s `In Array` and
`Weights` actually need.

Files changed (all under `Plugins/PinWright/Source/PinWright/Private/`):
- `Handlers/Audio/MetaSound/MetaSoundLiteralFromTypeName.h` / `.cpp` — array classification,
  element-type resolution, array-literal builder, array read-back fields.
- `Handlers/Audio/MetaSound/MetaSoundLiteralParams.h` — the `arrayValue` branch, its two new
  error emitters, and a missing-value message that names `arrayValue` for an array target.
- `Handlers/Audio/AudioAuthoringHandler.cpp` (`set_metasound_default`),
  `MetaSound/MetaSoundNodeInputDefaultHandler.cpp`, `MetaSound/MetaSoundVariableHandler.cpp`
  (both variable verbs) — `RPC_PARAM_OPT("arrayValue", "array", …)` plus a summary clause.
  Declaring it on all four is required: the shared helper reads the key, and an undeclared read
  is refused `UNKNOWN_PARAMS` by the dispatcher and caught by
  `infra.declared_params.HandlersOnlyReadDeclaredParams`.
- `Docs/wiki-src/audio.authoring.metasound_gotchas.md` — new "array-typed input or pin takes
  `arrayValue`" section.

Tests added (`Tests/Assets/TestMetaSoundLiteralGaps.cpp`, alongside the existing literal-gap
tests):
- `PinWright.audio.authoring.metasound_array_literal.ShapeFollowsTheDataType` — the pure builder
  contract, which is what BOTH default-setting verbs call: classification, element-name
  resolution, alias canonicalization through the array name, `Float:Array` from numbers, empty
  array, `WaveAsset:Array` from an asset path, and the failure directions (string into
  `Float:Array`, fractional into `Int32:Array`, array literal onto a scalar type, unresolvable
  object entry).
- `PinWright.audio.authoring.set_metasound_default.ArrayInputAcceptsArrayValue` — end to end
  through the RPC: create a `WaveAsset:Array` graph input, populate it with `arrayValue`, and read
  the result back off the DOCUMENT (not the response). Plus the two refusals — `objectValue` on an
  array input (the exact call this ticket filed; the message must now name `arrayValue`) and
  `arrayValue` on a scalar input — each followed by an assertion that the existing array default
  survived, i.e. nothing was written.

Reviewer verification: build `Array.Random Get (WaveAsset:Array)` in a real graph, bind its
`In Array` with `set_metasound_node_input_default {arrayValue: [wave1, wave2, wave3]}` and its
`Weights` with `{arrayValue: [0.5, 0.3, 0.2]}`, then confirm through `describe_metasound` and an
audition that the round-robin actually varies. NOT COMPILED OR RUN — a separate compile pass
follows.

## History
- `#1-filed` `OPEN` reporter — Found while adding per-shot round-robin to the weapon layers of the FPS build on EAContentExamples58 (UE 5.8), the standing rubric-row-6 defect the critic has scored low for three review rounds. `add_metasound_input` accepted `inputType: "WaveAsset:Array"` and created the input; `set_metasound_default` then refused every way of filling it, answering `INVALID_ASSET_TYPE ... accepts no object literal ... Use floatValue / intValue / boolValue / stringValue`, none of which can carry an array of object references. `set_metasound_node_input_default` exposes the same five scalar value parameters, so a node's `In Array` pin is equally unreachable. Since no "make array" node exists and every `Array.*` node consumes an array it cannot be handed, the whole family is dead from the RPC surface — including `Array.Random Get (WaveAsset:Array)`, which has a `No Repeats` input designed precisely for the round-robin being built. Verified the probe input was really created and then removed it cleanly with `remove_metasound_input`, so nothing was left behind. Workaround adopted: an arithmetic selector (`UE.RandomFloat` -> `UE.Multiply.Float` -> `Clamp.Clamp.Float` -> `UE.Subtract.Float` -> `UE.Multiply.Audio by Float` per variant), roughly 4 nodes for a fair 2-way switch versus 1 node for the array route, and about 16 for a 3-way — which is why this build ships two variants per layer rather than three. Asks for an `arrayValue` parameter on both default-setting verbs; note that `Array.Random Get`'s `Weights` pin is a `Float:Array` and so is unreachable too, making this the MetaSound-side twin of `B-cue-random-node-weights-zero-always-picks-first`.
- `#2-arrayvalue-through-the-shared-literal-helper` `IN-REVIEW` developer — Confirmed TRUE and implemented. Root cause is narrower than "no array param": `RegisterDataTypeArrayWithFrontend` registers `TArray<T>` without the `UClassToUse` template argument, so an array type's `ProxyGeneratorClass` is null and only `bIsProxyArrayParsable` is set, while `IsValidUObjectForDataType` tests `bIsProxyParsable` — hence the "accepts no object literal" refusal, and hence element class validation has to go through the element type NAME, which the registry never relates to the array type. Added `MakeArrayLiteralForMetaSoundType` to the existing `MetaSoundLiteralFromTypeName` helper (element form taken from the ARRAY type's own registry literal shape, object entries resolved and class-checked by reusing `MakeObjectLiteralForMetaSoundType` against the element type) and wired an `arrayValue` branch into the shared `BuildMetaSoundLiteralFromParams` — so `set_metasound_default`, `set_metasound_node_input_default`, `add_metasound_variable` and `set_metasound_variable_default` all gain it from one edit, rather than a parallel array path. Array and scalar are exclusive and the TARGET's declared type decides which applies, so both wrong-param directions are refused with a message naming the one that works. Also canonicalized array names through their element (`Int:Array` → `Int32:Array`) and added `arrayNum` / `objectPaths` to the literal read-back so an array write can be verified element by element. Two tests added; see the `## Fix` section above. Not compiled or run; a separate compile pass follows.
