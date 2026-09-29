---
id: F-pose-search-schema-channel-kinds
title: "pose_search.create_schema accepts only Position channels, so a usable Motion Matching schema (trajectory, velocity, heading) cannot be authored"
status: OPEN
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
