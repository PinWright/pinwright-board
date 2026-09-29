---
id: F-actor-batch-transform-lookat
title: "actor.set_transform / actor.nudge move one actor per call, and nothing aims an actor at a point or another actor"
status: IN-REVIEW
severity: Medium
category: feature
tags: [actor, set_transform, nudge, batch, lookAt, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# Batch actor transforms and `lookAt`

`actor.set_transform` and `actor.nudge` each took exactly one `actorName`. Moving N actors cost N
round trips, and `docs/wiki-src/level-building.md` documented "no verb moves many actors in one
call" as a known gap, steering callers to per-actor loops (the shape that wedged an editor for 168
minutes, `docs/rpc-design.md` §8). Aiming a camera, light or prop at a target required the caller
to compute the rotator by hand. The 2026-09-28 gap analysis against other Unreal MCP servers
listed both as cheap, high-value gaps.

**Fix:** both verbs accept an additive `actors[]` entry array (each entry carries its own actor
identity plus the single-form per-actor fields); `actor.set_transform` gains `lookAt` (a point
`{x,y,z}` or an actor identifier) plus `roll`. Single-actor calls keep their params and response
shape.

## History
- `#1-gap-analysis-filed` `OPEN` reporter — Filed from the 2026-09-28 gap analysis (plan task 8): batch transforms under one transaction with per-entry results, plus `lookAt` (point or actor, with `roll`) on `actor.set_transform`.
- `#2-batch-and-lookat-implemented` `IN-REVIEW` developer — New `Handlers/Actor/ActorBatchUtils.h` holds the shared two-phase driver: `ReadEntries` refuses a malformed batch (empty array, non-object entry, entry without identity, per-verb shape fault, single-form keys beside `actors[]`) with `INVALID_ARGUMENT` listing every faulty index before anything moves; `RunBatch` resolves and applies entries in order inside one `FScopedTransaction` and returns `{success, total, succeededCount, failedCount, results[] (index-aligned rows: single-form payload or errorCode/message/candidates), missing[], ambiguous[]}`, as an error with the first failure's code only when no entry applied. `ActorTransformHandler.cpp` / `ActorNudgeHandler.cpp` moved their single-actor bodies into `SetTransformApply` / `NudgeApply` (shared by both forms, so single-form responses are unchanged), declare `actors` via `RPC_PARAM_OPT_NESTED` (dispatcher refuses unknown entry keys), and make `actorName` optional (`ActorNameParamUtils::ActorNameParamOpt`); a call with neither returns `INVALID_ARGUMENT`. `lookAt` aims +X from the new location via `FRotationMatrix::MakeFromX`, applies `roll`, and is verified on the measured forward axis (`aimErrorDegrees`, `TRANSFORM_MISMATCH` above 0.1°); refused with `rotation`, `roll` without `lookAt`, self-aim, coincident target. Remaining raw error literals in both files converted to `ErrorCodes::ERR_*`; no new codes. Tests: `Tests/Actor/TestActorBatchTransform.cpp` (9 tests); `TestActorNameParamAlias` now expects `set_transform`'s actorName optional. Wiki: `actor.md` (new `actor.set_transform` section, nudge batch form), `level-building.md` (known-gap text replaced). Not compiled or run in this wave.
- `#3-listed-nested-gate-adopters` `IN-REVIEW` developer — Verification run `pw_gapwave_groups3.log` failed `PinWright.infra.dispatcher.NestedParamKeyGate.AdoptionSetIsRatcheted`: the two `actors` slots adopt the nested-key gate via `RPC_PARAM_OPT_NESTED` but were not in its adopter list. Added `actor.set_transform:actors` and `actor.nudge:actors` to `Tests/Infra/TestNestedParamKeyGate.cpp`, and both `actors` descriptions (plus `actor.md`) now state that any other entry key is refused with `UNKNOWN_NESTED_PARAMS`.
