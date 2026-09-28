---
id: B-bpir-inputkey-modifiers-dropped
title: "BPIR decompile renders InputKey events without their modifier flags, so a Ctrl+J binding reads as `entry key_pressed J()` in bpir.txt and the grammar cannot express Ctrl/Alt/Shift/Cmd at all"
status: OPEN
severity: High
category: bug
tags: [bpir, decompile, asset-dump, inputkey, k2node-inputkey, modifiers, round-trip, silent-wrong-data]
encounters: 1
lastSeen: 2026-09-28T09:32:00Z
---

# InputKey modifiers vanish from the BPIR text

`UK2Node_InputKey` carries four modifier bits next to `InputKey`: `bControl`, `bAlt`, `bShift`, `bCommand`
(`Engine/Source/Editor/BlueprintGraph/Classes/K2Node_InputKey.h:57-69`). The BPIR emitter ignores all of them.
`Decompiler/BpirTextEmitter.cpp:1868-1888` builds the entry line from
`FBpirInputKeyHelpers::FormatInputKeyAsBpirIdentifier(InputKeyNode->InputKey)` and the pressed/released pin,
nothing else:

```cpp
FString Kind = bIsReleased ? TEXT("key_released") : TEXT("key_pressed");
return AppendEntryPosition(FString::Printf(TEXT("entry %s %s()"), *Kind, *KeyName));
```

No file under `Compiler/` or `Decompiler/` references `bControl`/`bAlt`/`bShift`/`bCommand`, and the parser
(`Compiler/BpirParser.cpp:851-857`) has no syntax for a modifier, so this is a grammar gap as well as an emitter one.

## Observed

UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`. The asset dump
`asset-dumps/App/App/LevelBlueprints/B_LevelEditorCharacter/bpir.txt` shows `entry key_pressed Z()` (line 56),
`Y()` (61), `C()` (66), `V()` (71), `J()` (97) and `H()` (102). Live `blueprint.graph.find_nodes` on the same
Blueprint titles those nodes "Ctrl Z", "Ctrl Y", "Ctrl C", "Ctrl V", "Ctrl J", "Ctrl H". An agent reading the dump
concludes the level editor binds plain J/H/Z/..., which is wrong: a plain J press does nothing.

## Why High

Silent wrong data on the normal read path (the dump is the documented way to read a graph without opening it).
It also means a decompile -> edit -> `compile_bpir` round trip rebuilds these nodes as unmodified keys, and the
upsert identity in `Compiler/BpirCompiler.cpp:3714-3730` is `key_pressed <Key>` with no modifier, so `J` and
`Ctrl J` in one Blueprint would be treated as the same entry.

**Workaround:** read InputKey nodes with `blueprint.graph.find_nodes` (node titles include the modifier).
**Fix (proposed):** add modifier syntax to the entry line (e.g. `entry key_pressed Ctrl+J()` or
`entry key_pressed J(ctrl, shift)`), emit it from `BpirTextEmitter.cpp`, parse and apply it in the compiler, and
include the modifiers in the upsert identity. Bump the `bpir.txt` aspect version in
`AssetDumpCache.cpp::GetAspectVersion` in the same commit.

## History
- `#1-ctrl-modifier-missing-in-dump` `OPEN` reporter - Found while driving the PDS map editor in PIE: the dump said J/H toggled the tools, the live graph said Ctrl J / Ctrl H. Emitter at plugin `8fcc0b2a` confirmed to never read the modifier bits. Cheap this time (caught by a live `find_nodes` cross-check).
