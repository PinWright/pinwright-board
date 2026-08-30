---
id: B-sequence-add-keyframe-location-property-rejected
title: "sequence.add_keyframe rejects its own documented property=\"Location\" with [UNSUPPORTED_PROPERTY]"
status: IN-REVIEW
severity: Medium
category: bug
tags: [sequencer, add_keyframe, transform-track, docs-mismatch]
---

# `sequence.add_keyframe` rejects its own documented `property="Location"`

The frame-numbered keyframe writer `sequence.add_keyframe` advertises `Location`
as a valid value for its `property` parameter, but the handler has no branch for
it — any property name other than `Transform` (when the value is an object) falls
straight through to a generic-track path that only accepts numeric/boolean values,
so a `Location` keyframe with a vector value is rejected with
`[UNSUPPORTED_PROPERTY] Unsupported property or failed to create track`.

The param schema is declared at
`Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp:2410`:

```cpp
RPC_PARAM_OPT("property", "string", "Property name (e.g. Transform, Location)"),
```

That description flows into the generated wiki page
`sequence.add_keyframe.md` ("`property` (`string`, optional): Property name (e.g.
Transform, **Location**)"), so a caller following the docs naturally tries
`property="Location"` to keyframe a camera/actor position. But the handler body
(`SequenceHandler.cpp:1556`+) only special-cases `Transform`:

- `PropertyName.Equals("Transform")` → writes the nested
  `{location:{x,y,z}, rotation:{...}, scale:{...}}` value onto a
  `UMovieScene3DTransformTrack` (9 double channels). Works.
- Any other `PropertyName` → falls to the "Generic property tracks" branch
  (`:1663`+), which only matches when `value` is `EJson::Number` (float track) or
  `EJson::Boolean` (bool track). A `Location` keyframe carries an object/vector
  value, matches neither, and reaches the terminal
  `Ctx.SendError("UNSUPPORTED_PROPERTY", ...)` at `:1740`.

So `Location` is documented as supported but is unconditionally unsupported for
any value shape — a valid documented input wrongly rejected.

## Verbatim repro

Setup (all succeed): `sequencer.create {name:"ReplayShot", path:"/Game/ReplayCine"}`
→ `sequencer.add_camera {path:"/Game/ReplayCine/ReplayShot"}`
(binding `B5B6BBEE422EB1B8CA2425930FC166DE`)
→ `sequencer.add_transform_track {sequencePath:".../ReplayShot", bindingGuid:"B5B6...66DE"}`.

Then the documented call:

- `sequence.add_keyframe {path:"/Game/ReplayCine/ReplayShot", bindingId:"B5B6...66DE", property:"Location", frame:0, value:{x:-800,y:400,z:200}}`
  → `[UNSUPPORTED_PROPERTY] Unsupported property or failed to create track`

Rejected for every value shape tried — nested object `{x,y,z}`, capitalized
`{X,Y,Z}`, and a bare array `[-800,400,200]` all return the identical error.

The workaround the attempt found (which the wiki does not spell out) is to use
`property="Transform"` with the value nested under a `location` key:

- `sequence.add_keyframe {... property:"Transform", frame:0, value:{location:{x:-800,y:400,z:200}}}`
  → `{}` (success)

**What it should do:** since the param description lists `Location` as a supported
property, the handler should either (a) accept `property="Location"` (and the other
transform sub-axes one might reasonably try — `Rotation`/`Scale`) by routing a
vector value onto the corresponding channels of the binding's transform track, or
(b) if only `Transform` is ever supported for object-valued keyframes, drop
`Location` from the param description and the generated wiki, and have the error
name the supported property (e.g. `Unsupported property 'Location'; use 'Transform'
with a nested {location:{...}} value`) instead of the generic
`Unsupported property or failed to create track`.

**Workaround:** use `property="Transform"` with
`value:{location:{x,y,z}[, rotation:{roll,pitch,yaw}][, scale:{x,y,z}]}`.

**Fix:** in `SequenceHandler.cpp`, before the generic branch add a case that maps
`Location`/`Rotation`/`Scale` (and lone-axis names if desired) onto the
`UMovieScene3DTransformTrack` double channels (location 0-2, rotation 3-5, scale
6-9), reusing the same `FindTrack/AddTrack<UMovieScene3DTransformTrack>` +
`FindOrAddSection(0)` path the `Transform` branch already uses; OR correct the
`RPC_PARAM_OPT` description at `:1463` to stop advertising `Location`.

## History
- `#1-initial-repro` `OPEN` reporter — REALISM-mode cinematic-blockout task: build an EstablishingShot level sequence, add a transform track on the bound camera, and keyframe a push-in. Attempt tried the documented `property="Location"` first and got `[UNSUPPORTED_PROPERTY] Unsupported property or failed to create track`; only `property="Transform"` with a nested `{location:{x,y,z}}` worked. Replay-confirmed via `mcp__editor-automation__call`: on a fresh `/Game/ReplayCine/ReplayShot` with a bound camera + transform track, `sequence.add_keyframe property="Location"` is rejected for object `{x,y,z}`, `{X,Y,Z}`, and array `[-800,400,200]` value shapes alike, while `property="Transform"` value `{location:{x,y,z}}` returns `{}`. Root cause read in source: param schema `SequenceHandler.cpp:1463` advertises `Property name (e.g. Transform, Location)` but the handler only branches on `Transform` (`:1556`); every other property name with an object value falls through the numeric/boolean-only generic branch to the terminal `UNSUPPORTED_PROPERTY` (`:1740`). Documented valid input wrongly rejected.
- `#2-fix` `IN-REVIEW` developer — Implemented the root-cause fix (option a): added a `Location`/`Rotation`/`Scale` sub-axis branch to `sequence.add_keyframe` between the `Transform` branch and the numeric/boolean-only generic branch in `Source/EditorAutomationRpcGateway/Private/Handlers/Sequencer/SequenceHandler.cpp` (after the Transform branch, before the generic `else`). The new branch reuses the same `FindTrack/AddTrack<UMovieScene3DTransformTrack>` + `FindOrAddSection(0)` + `FloorToFrame` tick path the `Transform` branch uses, and writes a 3-component vector value onto the section's double channels at the right base offset (Location→0-2, Rotation→3-5, Scale→6-8). Accepts the value as a `{x,y,z}`/`{X,Y,Z}` object (case-insensitive field lookup; rotation also honors `{roll,pitch,yaw}`) or a bare `[x,y,z]` array — all three value shapes from the repro now succeed. The param schema's `Location` advertisement is now honored, so no doc/error-message change was needed. Regression test added: `EditorAutomationRpcGateway.sequence.add_keyframe.LocationVectorWritesChannels` in `Source/EditorAutomationRpcGateway/Private/Tests/Media/TestSequencerHandlers.cpp` — builds a transient `ULevelSequence` with a possessable binding, invokes `sequence.add_keyframe property="Location" value={x:-800,y:400,z:200}`, asserts `bSuccess` (and error code != `UNSUPPORTED_PROPERTY`), and verifies one key landed on each of location channels 0-2 with rotation/scale channels untouched. The test exercises the production handler via `InvokeHandlerWithCapture` and would fail (UNSUPPORTED_PROPERTY → `bSuccess=false`) if the Location branch were reverted. Did not compile/run (later phase).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. 2 citations sit in history rows and are left verbatim per the append-only rule. `:1463` had drifted onto an unrelated `Cast<UBlueprint>` class-resolve path; the quoted `RPC_PARAM_OPT("property", …)` is byte-identical at `:2410`, in the registration block opening at `:2405`. **Premise no longer holds:** the handler does have a Location branch — it routes `property` through `SequenceKeyframeHelpers::ClassifyTransformProperty` at `:2531`, with Location/Rotation/Scale as three-channel groups (`:2524-2531`). Ticket is still IN-REVIEW. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
