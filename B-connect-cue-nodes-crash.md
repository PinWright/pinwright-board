---
id: B-connect-cue-nodes-crash
title: "audio.authoring.connect_cue_nodes crashes the editor via a SoundCueGraph InputPins==ChildNodes assertion when wiring cue nodes"
status: OPEN
severity: Critical
category: bug
tags: [audio, sound-cue, connect-cue-nodes, crash, assertion, graph-pins]
encounters: 1
lastSeen: 2026-07-01T08:26:03.7264932+03:00
---

# connect_cue_nodes trips a fatal SoundCueGraph pin-count assertion and kills the editor

`audio.authoring.connect_cue_nodes` mutates the source node's `ChildNodes`
array directly and then calls `USoundCue::LinkGraphNodesFromSoundNodes()`. That
engine routine walks each `USoundNode`'s paired EdGraph node and asserts that
the graph node's input-pin count matches the node's `ChildNodes` count. Because
the handler resizes/assigns `ChildNodes` **without first reconstructing the
EdGraph node's input pins** to match the new child count, the invariant is
violated and the whole editor dies on a fatal `check()` — a hard crash, not a
recoverable error. The MCP connection is forcibly closed (WinError 10054) and
every subsequent probe is refused (WinError 10061). Inputs are valid and
documented, so this is a tool crash, not bad-input rejection.

## Repro (verbatim, from the attempt that crashed the host)

1. `audio.authoring.create_sound_cue` → `Cue_GlassFootsteps` at
   `/Game/ExampleContent/Audio/Cues/Cue_GlassFootsteps` (looping=false, vol=0.7;
   this seeds a root Modulator node named `Modulator_0`).
2. `audio.authoring.add_cue_node` random (save=false) → `Random_0`.
3. `audio.authoring.add_cue_node` wave_player × 4 (Glass01..Glass04).
4. `audio.authoring.connect_cue_nodes`
   `{ assetPath: "/Game/ExampleContent/Audio/Cues/Cue_GlassFootsteps",
      sourceNodeId: "Modulator_0", targetNodeId: "Random_0", childIndex: 0 }`
   → **editor crashed** on the very first connect. Retried connect and all
   follow-up probes (`get_audio_info`, `describe_sound_cue`) returned
   `Editor not reachable ... ([WinError 10054] ... forcibly closed)` then
   `[WinError 10061] ... connection refused`. The graph was never wired and
   nothing could be verified.

Trigger call / self-report: the graph had a mix of nodes whose EdGraph input-pin
layout diverges from their `ChildNodes` count (a Modulator seeded by
create_sound_cue plus a Random + wave-players added via add_cue_node with no pin
sync); the first `connect_cue_nodes` forced a `LinkGraphNodesFromSoundNodes()`
relink over that mismatched state.

## Verbatim assert + callstack (ground truth from the crash dump)

Crash folder: `Saved/Crashes/UECC-Windows-E9E2955A458534FD2E1865B6AFFB9953_0000`
(CrashType = Assert). From `CrashContext.runtime-xml` / `EAContentExamples57.log`:

```
Assertion failed: InputPins.Num() == SoundNode->ChildNodes.Num() [File:D:\build\++UE5\Sync\Engine\Source\Editor\AudioEditor\Private\SoundCueGraph.cpp] [Line: 61]

UnrealEditor_Core!FDebug::CheckVerifyFailedImpl2() [.../Misc/AssertionMacros.cpp:745]
UnrealEditor_AudioEditor!FSoundCueAudioEditor::LinkGraphNodesFromSoundNodes() [.../Editor/AudioEditor/Private/SoundCueGraph.cpp:61]
UnrealEditor_Engine!USoundCue::LinkGraphNodesFromSoundNodes() [.../Runtime/Engine/Private/SoundCue.cpp:898]
UnrealEditor_PinWright!AutoHandler_309_() [X:\...\Plugins\PinWright\Source\PinWright\Private\Handlers\Audio\AudioAuthoringHandler.cpp:657]
UnrealEditor_PinWright!FRpcDispatcher::DrainAutoRegistrations'::`6'::<lambda_1>::operator()() [.../Dispatch/RpcDispatcher.cpp:296]
UnrealEditor_PinWright!FRpcDispatcher::ProcessRequest() [.../Dispatch/RpcDispatcher.cpp:479]
UnrealEditor_PinWright!FMcpTransport::Start'::`2'::<lambda_1>::operator()() [.../Transport/McpTransport.cpp:566]
UnrealEditor_HTTPServer!FHttpConnection::ProcessRequest() [.../Online/HTTPServer/Private/HttpConnection.cpp:213]
```

## Guilty source (verbatim)

`Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp:650-657`:

```cpp
    if (ChildIndex >= SourceNode->ChildNodes.Num())
    {
        SourceNode->ChildNodes.SetNum(ChildIndex + 1);
    }
    SourceNode->ChildNodes[ChildIndex] = TargetNode;

    Cue->LinkGraphNodesFromSoundNodes();
```

The handler grows/assigns `ChildNodes` and immediately relinks. It never
reconstructs the paired `USoundCueGraphNode`'s input pins (e.g. via the node's
`CreateInputPins()` / `ReconstructNode()` or a full graph rebuild) so that
`InputPins.Num()` tracks the new `ChildNodes.Num()`. When any node in the cue
has a pin count that diverges from its child count at relink time,
`FSoundCueAudioEditor::LinkGraphNodesFromSoundNodes()` fires the fatal
`check(InputPins.Num() == SoundNode->ChildNodes.Num())`.

## What it should do

Keep the EdGraph node's input pins in sync with `ChildNodes` before relinking —
reconstruct/resize the source (and any affected) SoundCueGraphNode's input pins
to match the new child count, or rebuild the graph node representation — so the
engine invariant holds. If the requested slot/node type genuinely cannot accept
the child, validate up front and return a clean domain error (e.g.
`GRAPH_PIN_MISMATCH` / `INVALID_CHILD_INDEX`) instead of tripping a fatal engine
`check()`. Under no valid input should a graph-wiring RPC be able to crash the
host editor.

severity rationale: impact=crash × reach=rare -> Critical

## History
- `#1-initial-repro` `OPEN` reporter — `audio.authoring.connect_cue_nodes` crashed the whole editor on the first connect of a `Modulator_0 -> Random_0` (childIndex 0) wire in `Cue_GlassFootsteps`. The crash is a fatal engine assertion `InputPins.Num() == SoundNode->ChildNodes.Num()` at `SoundCueGraph.cpp:61`, reached from the handler's `Cue->LinkGraphNodesFromSoundNodes()` call at `AudioAuthoringHandler.cpp:657` after it mutates `SourceNode->ChildNodes` (lines 650-654) without reconstructing the EdGraph node's input pins. Ground truth: crash dump `Saved/Crashes/UECC-Windows-E9E2955A458534FD2E1865B6AFFB9953_0000` (CrashType=Assert) + editor log; MCP forcibly closed (WinError 10054) then connection-refused (WinError 10061) on every follow-up probe, so the wire never completed and nothing could be verified. Fix: sync graph-node input pins to `ChildNodes` before relinking, or reject invalid slots with a clean error instead of a fatal `check()`.
