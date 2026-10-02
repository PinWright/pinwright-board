---
id: F-pose-search-schema-channel-kinds
title: "pose_search.create_schema accepts only Position channels, so a usable Motion Matching schema (trajectory, velocity, heading) cannot be authored"
status: DONE
severity: Medium
category: feature
tags: [pose-search, motion-matching, create-schema, feature-channel, authoring, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# pose_search.create_schema supports only the Position channel

`AddChannelFromSpec` (`Source/PinWrightPoseSearch/Private/Handlers/PoseSearch/PoseSearchHandler.cpp:334-403`,
plugin HEAD `71c91649`) normalises the channel `type`/`kind` and refuses anything but `position`
with `UNSUPPORTED_CHANNEL` "Supported channels: Position." (`:357-362`), then builds a
`UPoseSearchFeatureChannel_Position`. The param doc says the same (`:718`).

A Motion Matching schema that selects sensible poses needs at least a Trajectory channel (future
and past root samples) and usually Velocity and Heading channels; Position alone matches bone
placement with no notion of where the character is going. The accepted proposal in
`F-pose-search-database-authoring` (`#1`) listed Position / Velocity / Trajectory / Heading / Phase;
the shipped verb delivered one of the five, and that ticket was closed on a Position-only test.

UE 5.8 ships public headers for `Curve`, `Distance`, `Group`, `Heading`, `Padding`,
`PermutationTime`, `Phase`, `Pose`, `Position`, `SamplingTime`, `TimeToEvent`, `Trajectory` and
`Velocity` channels (`Engine/Plugins/Animation/PoseSearch/Source/Runtime/Public/PoseSearch/PoseSearchFeatureChannel_*.h`).
On 5.3 to 5.5 the concrete channel headers are Private, so per-kind typed code needs the reflection
route described in `B-pose-search-gate-private-header`.

Engines: all supported.

**Fix:** accept at least `trajectory`, `velocity`, `heading` and `pose` (the compound Pose channel
covers per-bone position/velocity sets), and preferably any `UPoseSearchFeatureChannel` subclass by
class name with its properties set from the spec object through reflection, so new engine channel
kinds need no handler change. Echo each created channel's class and resolved settings in the
response. Add the new kinds to the schema test.

**Related:** `F-pose-search-database-authoring`, `B-pose-search-gate-private-header`.

## History
- `#1-position-only` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis and re-verified at plugin HEAD `71c91649`: `AddChannelFromSpec` rejects every non-`position` kind (`PoseSearchHandler.cpp:357-362`). Filed as a new feature ticket rather than reopening `F-pose-search-database-authoring`, which is DONE and whose delivered verb works as far as it goes. Severity Medium: a hard blocker for a working Motion Matching setup (High-or-Medium class), bumped down for rare reach.
- `#2-any-channel-class` `IN-REVIEW` developer - `pose_search.create_schema` accepts any concrete `UPoseSearchFeatureChannel` subclass by kind (class name minus `PoseSearchFeatureChannel_`, case/`_`/`-`/space-insensitive; `type` or `kind` key; object paths reduce to their last segment), so Trajectory/Velocity/Heading/Pose and every other 5.8 kind work with no per-kind code. Every other spec key sets the channel's `CPF_Edit` property of that name via `ApplyJsonValueToProperty` (structs as objects, struct arrays such as `samples` / `sampledBones` as arrays of objects, enums by name, bitmask flags as ints; FBoneReference also takes a bare bone name; `boneName`/`originBoneName` aliases kept). Unknown kind -> `UNSUPPORTED_CHANNEL` listing the engine's kinds; unknown setting or bad value -> `INVALID_ARGUMENT` listing settable properties (previously unknown keys were silently dropped). Channels are now built under the transient package before the schema exists, so a refused spec leaves no half-created schema. Response adds `channels[]` `{kind, className, settings}` (settings = every editable property, resolved). Group channels are creatable but their instanced `subChannels` are not authorable. Files: `Source/PinWrightPoseSearch/Private/Handlers/PoseSearch/PoseSearchHandler.cpp`, `Source/PinWrightPoseSearch/Private/Tests/Gameplay/TestPoseSearchHandlers.cpp`, `docs/wiki-src/pose_search.md`, `CHANGELOG.md`. Test `PinWright.pose_search.CreateSchemaChannelKinds`: creates Trajectory (explicit samples), Velocity (via `kind`, weight 2.5), Heading (`headingAxis` Y), Position (`boneName` alias, sampleTimeOffset 0.25) and Pose (one sampled bone), asserts each lands in the finalized schema with the set values read back by reflection and the echo's kind/className/settings, then asserts `NotAChannel` -> UNSUPPORTED_CHANNEL and an unknown setting -> INVALID_ARGUMENT with no schema object left behind. Compile-checked on 5.8; not yet run.
- `#3-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). `PinWright.pose_search.CreateSchemaChannelKinds` passed in w23-final (no skip): Trajectory (explicit samples), Velocity (via `kind`, weight 2.5), Heading (`headingAxis` Y), Position (`boneName` alias, `sampleTimeOffset` 0.25) and Pose (one sampled bone) each land in the finalized schema with the values read back by reflection and echoed in `channels[] {kind, className, settings}`; `NotAChannel` -> `UNSUPPORTED_CHANNEL` and an unknown setting -> `INVALID_ARGUMENT` leave no schema behind. The Fix (at least trajectory, velocity, heading, pose; any subclass by name via reflection; echo; test) is met on 5.8. Limits: Group sub-channels are not authorable; UE 5.3-5.5 are unverified here and depend on `B-pose-search-gate-private-header` (stays IN-REVIEW).
