---
id: B-decompile-drops-timeline-tracks
title: "BPIR decompile emits `timeline Name()` with no tracks, so decompile then recompile silently deletes every Timeline track"
status: IN-REVIEW
severity: Critical
category: bug
tags: [bpir, decompiler, compile, timeline, k2node-timeline, timeline-template, round-trip, data-loss, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# BPIR decompile drops Timeline tracks

A Blueprint with a Timeline cannot survive a BPIR round trip. The decompiler writes the Timeline as a
bare name, the compiler rebuilds it from that text, and every track the Timeline had (float, vector,
linear color and event tracks, with their curves) is gone. Nothing reports the loss: the decompile
looks complete and the recompile succeeds.

Evidence at plugin HEAD `71c91649`:

- **Decompile side.** `Source/PinWright/Private/Decompiler/BpirTextEmitter.cpp:2923-2938`,
  `FBpirTextEmitter::EmitTimeline`, reads only `TimelineNode->TimelineName` and returns
  `%result = timeline Name() [exec targets]`. It never touches the node's `UTimelineTemplate`: no
  tracks, no keys, no length, no loop / autoplay / replicated flags. A plugin-wide grep for
  `FloatTracks|VectorTracks|EventTracks|LinearColorTracks` outside tests finds one hit, the compiler
  line below; nothing serializes Timeline data anywhere (asset-dump `bpir.txt` included).
- **Recompile side.** Replacing an event body collects the old subgraph and calls
  `FBlueprintEditorUtils::RemoveNode` on each node (`Source/PinWright/Private/Compiler/BpirCompiler.cpp:3256-3260`).
  For a Timeline node that reaches `UK2Node_Timeline::DestroyNode`
  (`Engine/Source/Editor/BlueprintGraph/Private/K2Node_Timeline.cpp:204-218`), which removes the
  `UTimelineTemplate` and renames it into the transient package. The rebuilt node then gets a fresh,
  empty template from `FCodeNodeEmitter::CreateTimelineNode`
  (`Source/PinWright/Private/Compiler/CodeNodeEmitter.cpp:613-627`), and the decompiled text carries no
  track args to refill it. Any downstream wire that read a track output pin has no pin to bind to.

Compile-side fragility found on the same path (`BpirCompiler.cpp:6608-6634`):

- Only `float_curve(...)` args become tracks. Vector, linear color and event tracks have no syntax,
  so even a fixed emitter could not express them today.
- The template to fill is picked as `TargetBlueprint->Timelines.Last()`, not the template of the node
  just created. `CreateTimelineNode` discards the return value of
  `FBlueprintEditorUtils::AddNewTimeline`, which returns `nullptr` and only logs when a template with
  that name already exists (`Engine/Source/Editor/UnrealEd/Private/Kismet2/BlueprintEditorUtils.cpp`,
  `AddNewTimeline`). In that case the new node binds to the existing template by name while the
  `float_curve` tracks are appended to whichever template is last in the array, which can be a
  different Timeline.

Engines: all supported (5.3 to 5.8); the defect is in plugin code.

**Fix:** emit the full template from `EmitTimeline`: length, loop / autoplay / replicated /
ignore-time-dilation flags, and every float, vector, linear color and event track with its keys and
interpolation. Extend the compiler to parse the same forms. Resolve the template with
`Blueprint->FindTimelineTemplateByVariableName(Node->TimelineName)` (or the `AddNewTimeline` return
value) instead of `Timelines.Last()`, and fail loudly when the name collides. Add a round-trip test:
a Blueprint with one track of each kind, decompile, recompile in replace mode, compare templates.
Until then, the decompiler should at least refuse or warn when a Timeline template has tracks it
cannot emit, instead of producing text that deletes them.

**Related:** `B-orphan-sweep-deletes-isolated-timeline` (another path that destroys a Timeline
template), `B-bpir-timeline-multiline-example-unparseable` (Timeline syntax docs).

## History
- `#1-decompile-drops-tracks` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis and re-verified from source at plugin HEAD `71c91649`. `EmitTimeline` (`BpirTextEmitter.cpp:2923-2938`) emits `timeline Name()` from the node name alone; a replace-mode recompile removes the old node (`BpirCompiler.cpp:3256-3260`), whose `DestroyNode` discards the template (`K2Node_Timeline.cpp:204-218`), and rebuilds an empty one (`CodeNodeEmitter.cpp:613-627`). Compile side supports only `float_curve` and targets `Timelines.Last()` (`BpirCompiler.cpp:6615-6617`) while ignoring the `AddNewTimeline` return. Not reproduced live (it would destroy the tracks of a real asset); the source path is unambiguous. Severity Critical: a write that loses asset data with no signal.
- `#2-template-text-both-directions` `IN-REVIEW` developer - New `Compiler/BpirTimelineText.{h,cpp}` holds one grammar for both directions: `FormatArgs` (decompile) and `ParseArgs` + `Apply` (compile). `FBpirTextEmitter::EmitTimeline` now resolves the node's template (`FindTimelineTemplateByVariableName`) and emits `timeline Name(<settings>, <tracks>)`: `length`, `length_mode`, `autoplay`, `loop`, `replicated`, `ignore_time_dilation` when they differ from a new template, then every track in display order as `float_curve(...)`, `vector_curve(x(...), y(...), z(...))`, `color_curve(r(...), g(...), b(...), a(...))` or `event_curve(...)`, keys as `(t, v[, interp[, tangentMode, arrive, leave[, weightMode, arriveWeight, leaveWeight]]])` with shortest round-trip floats, external curve tracks as a quoted asset path (loaded through `LoadObjectChecked`). `EBpirOpcode::Timeline` in `BpirCompiler.cpp` parses all args before creating anything (malformed key/channel, duplicate track, unknown setting and unloadable curve are compile errors), `FCodeNodeEmitter::CreateTimelineNode` calls `AddNewTimeline` first and returns the template, creating no node when it returns null; the compiler reports "already exists" (name taken) or "does not support timelines" instead of binding to another template, then applies the spec to that template (curves outered to the generated class with `RF_Public`, as the Timeline editor does), `AddDisplayTrack` per track (the old path never did, so track pins did not exist until reload) and `ReconstructNode`. `Timelines.Last()` and `ParseFloatCurveKeyframes` are gone. `bpir.txt` aspect bumped 9 -> 10. Tests: `PinWright.bpir.timeline_tracks.{FloatTrackRoundTrips,VectorTrackRoundTrips,ColorTrackRoundTrips,EventTrackRoundTrips,SettingsRoundTrip,ReplaceModeRecompileKeepsTracks,DuplicateNameIsRejected,MalformedArgsAreRejected}` (`Tests/Bpir/TestBpirTimelineTracks.cpp`); compile-checked with `-SingleFile`, not yet run. Not carried: curve default value, pre/post-infinity extrapolation, timeline tick group (documented in `bpir.instructions` section 2.6). An external-curve track is covered only in the failure direction.
- `#3-explicit-tangents-kept` `IN-REVIEW` developer - Verifier run (`Saved/Logs/pw_gapwave_groups.log`) failed `timeline_tracks.FloatTrackRoundTrips`: key `(1, 1, cubic, user, 2, -3)` compiled with arrive/leave 0. Root cause: `Apply` wrote keys through `FRichCurve::SetKeys`, which runs `AutoSetTangents`; that zeroes both tangents of any key whose previous key is constant (and the leave tangent of a key after one), user and break keys included, so text-stated tangents were overwritten on compile (the round-trip comparison could not see it because source and rebuild lost them identically). Fix: `SetCurveKeys` in `Compiler/BpirTimelineText.cpp` calls `SetKeys`, then writes the parsed arrive/leave back for every key whose tangent mode is not `RCTM_Auto`; auto keys stay engine-derived. Same helper serves float, event, vector and color channels. Also: `PreloadExternalClasses` (`BpirCompiler.cpp`) no longer feeds timeline args to `ResolveUClass`, which logged `Failed to find object 'Class last_keyframe'` / `'Class maybe'`; the `'Class true'` warnings in the same tests are pre-existing (any `true` literal in any instruction, e.g. `PrintString` defaults in decompiled text, reaches that pass; they also appear in `round_trip.While`, `split_input.*`) and are not changed here. `-SingleFile` compile-check of both files: `Result: Succeeded`.
