---
id: E-ground-not-found-when-probe-starts-below-the-ground
title: "`GROUND_NOT_FOUND` with `overLandscape: true` and `supportedColumns: 0` can only mean the probe started under the terrain, but the message names neither that cause nor `probeLift`, the parameter that fixes it"
status: OPEN
severity: Medium
category: ergonomic
tags: [spatial, ground_actors, ground_instances, verify_grounding, probe-lift, error-message, error-code, actionable-hint, buried-actor, level-building]
encounters: 1
costly: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The response already knows why it failed and does not say so

`spatial.ground_instances` over a 630-instance scatter laid on a Z-600 plane, under terrain whose
surface sits between Z 1150 and Z 1750, returned `GROUND_NOT_FOUND` for **42** instances. Every one
of those 42 rows carried the same pair of measurements:

    reasonCode: "GROUND_NOT_FOUND"
    overLandscape: true
    supportedColumns: 0

The remedy was one parameter: `surface: {probeLift: 4000}`, after which **all 630 instances placed**.
Nothing else changed — same surface preset, same region, same meshes.

## The signature is unambiguous, which is what makes the message a defect

`overLandscape` is set only when the trace found nothing, and only from the landscape heightfield —
`GroundPlacementUtils.cpp:1079-1085`, inside `if (Report.SupportedColumns == 0)`:

```cpp
const TOptional<double> LandscapeZ = ProbeLandscapeHeight(World, Origin.X, Origin.Y);
Report.bOverLandscape = LandscapeZ.IsSet();
```

So `overLandscape: true` with `supportedColumns: 0` states, in the handler's own two facts: *the
landscape reports a height at this XY, and the downward trace hit nothing.* A downward probe over
terrain that reports a height can miss it for exactly one reason — it started below it. There is no
second interpretation to disambiguate, and the handler has already computed both halves before it
picks its message.

The message it picks is the generic one. `EvaluateContact`, `GroundPlacementUtils.cpp:707-731`,
branches on `bOverLandscape` and uses it only for the *negative* case:

```cpp
else if (Report.bOverLandscape.IsSet() && !Report.bOverLandscape.GetValue())
{
    Report.FailReasonCode = ErrorCodes::ERR_GROUND_NOT_FOUND;
    Report.FailReason = TEXT("No ground under any part of this actor, and the landscape "
                             "reports no height at its footprint either - it is off the "
                             "terrain entirely.");
}
else
{
    Report.FailReasonCode = ErrorCodes::ERR_GROUND_NOT_FOUND;
    Report.FailReason = TEXT("No accepted ground surface under any sampled column of this "
                             "actor's footprint.");
}
```

The `false` branch is written exactly as this ticket asks: it names the cause (*off the terrain
entirely*) and it is distinguishable. The `true` branch — the one that fired 42 times — falls into
`else` and returns a sentence that describes the symptom and stops. Both branches also share one
error code, so a caller cannot separate them programmatically either.

## The parameter that fixes it is undiscoverable from the wiki

The probe start is `GroundPlacementUtils.cpp:943`:

```cpp
const double ProbeStartZ = TopZ + FMath::Max(Surface.ProbeLiftCm, 0.0);
```

`ProbeLiftCm` defaults to `DefaultProbeLiftCm = 500.0` (`GroundPlacementUtils.h:76`, field at
`:147`), parsed from `surface.probeLift` / `surface.probe_lift` at `GroundPlacementUtils.cpp:413-418`
and again at `GroundPlacementHandler.cpp:295-300`. 500 cm of lift over a plane 550-1150 cm below the
terrain is not enough, which is the whole failure.

**The source knows this and the wiki does not.** `GroundPlacementUtils.h:70-74` states the case in
full — *"an actor BURIED in geometry has its ground above its underside, and a probe that starts at
the underside can never see it. That is the case that left rocks embedded in 1700 uu cliff faces
looking correct until the cliffs flattened"* — and `LevelAuditHandler.cpp:71-72` acts on it, pre-
seeding `AuditRpcProbeLiftCm = 100000.0` for exactly this reason. But in `Docs/wiki-src/spatial.md`
`probeLift` appears **once**, at `:270`, as a bare key inside the `surface` schema list, with no
default, no units and no statement of what it is for. `spatial.md:324` explains `overLandscape:
false` and says nothing about `overLandscape: true`. Recovering the 630 instances took a source dive,
not a doc read.

## What it should do

Register a distinct `GROUND_ABOVE_PROBE_START` in `Handlers/ErrorCodes.h` and emit it from the
`else` branch at `GroundPlacementUtils.cpp:725-729` when `bOverLandscape` is `true`, with a message
that names the three things the handler already holds: the probe start Z it used, the landscape Z it
found, and `probeLift` as the parameter to raise.

**The registration is not optional and not a follow-up.** `ErrorCodes.h:14-15` — *"Adding a code:
declare a new ERR_<CODE> constant here first, then reference it from the handler"* — and
`PinWright.core.error_codes.AllEmittedCodesAreRegistered`
(`Source/PinWright/Private/Tests/Core/TestErrorCodeRegistry.cpp:460-462`, purpose stated at `:6-7`)
is a **source scan** that fails any emitted code with no `ERR_<CODE>` entry. Its emission scan
follows codes reaching `SendError` through a variable (`:26-32`), which is precisely how this one
travels — assigned to `Report.FailReasonCode` and forwarded later — so the constant must land in the
same edit or the suite goes red. `GROUND_ABOVE_PROBE_START` currently has zero matches anywhere in
`Source/`.

Two smaller things belong in the same change, because the source already carries the knowledge and
only the caller-facing surface is missing it:

- **The convention this violates is written down.** `Docs/lessons.md:9`: *"generic messages like
  'Could not resolve value X' cause misdiagnosis — the caller blames the tool instead of their
  input. Always include diagnostic context (what was searched, what class was checked, actionable
  suggestions ...) in errors returned to MCP callers."* The sibling branch two lines up honours it;
  this one does not.
- **Document `probeLift` in `spatial.md`** beyond its appearance as a schema key — default `500` cm,
  measured up from the actor's top, and the buried-actor case it exists for. The `level.audit` path
  already pre-seeds `100000`; a caller of `spatial.ground_*` has no way to learn that is even a
  thing to consider.

## Distinct from

- `B-verify-grounding-maxgap-false-fail` (DONE, High) — the precedent for this shape and the reason
  the bar here is a message and not a number. There, `spatial.verify_grounding` reported truthful
  measurements (`minGapCm: -48.43`, `coverage: 1`, `contactPoints: 4`) under a verdict the numbers
  contradicted. Here the numbers and the verdict agree; what is missing is the sentence that turns
  two correct facts into an action. Same family (these verbs answer honestly and unhelpfully),
  strictly lesser severity — no false verdict is produced.
- `B-ground-probe-hits-hull-not-render` — the probe hits the *wrong* surface (a collision hull
  rather than the render mesh). This is the probe hitting *nothing*, because it started underneath
  everything. Different failure, different measurement, and a fix for either leaves the other.
- `E-ground-preset-excludes-only-foliage-actors` — the surface *filter* rejecting hits. That case is
  already separately coded (`GROUND_HITS_ALL_REJECTED`, `GroundPlacementUtils.cpp:711-719`) and is
  not what fired here: `rejectedSurfaceActorCount` was 0 on all 42 rows.

## Dedup

Searched the whole board (all statuses) for `GROUND_NOT_FOUND`, `probeLift`, `probe_lift` and
`overLandscape`: **zero files match any of the four**. Nothing on the board owns this error's
message, this parameter, or this measurement pair.

## History
- `#1-42-instances-with-the-signature` `OPEN` reporter — Measured, not inferred. `spatial.ground_instances` on a 630-instance scatter laid at Z 600, under terrain at Z 1150-1750: 42 instances returned `GROUND_NOT_FOUND`, every one with `overLandscape: true` and `supportedColumns: 0`; `surface: {probeLift: 4000}` then placed all 630 with no other change. The pair is diagnostic by construction — `bOverLandscape` is assigned only inside `if (Report.SupportedColumns == 0)` from the landscape heightfield (`GroundPlacementUtils.cpp:1079-1085`), so `true` there means the landscape has a height the downward trace never saw, which a probe starting above the terrain cannot produce. `EvaluateContact` (`:707-731`) already branches on the flag but spends it only on the negative case ("it is off the terrain entirely", `:717-722`); the positive case falls to a generic `else` (`:725-729`) sharing the same error code, so the caller can separate them neither by text nor by code. Probe start is `TopZ + Surface.ProbeLiftCm` (`:943`), default `DefaultProbeLiftCm = 500.0` (`GroundPlacementUtils.h:76`, field `:147`), parsed at `:413-418` and `GroundPlacementHandler.cpp:295-300`. The knowledge exists in source (`GroundPlacementUtils.h:70-74`, the buried-rocks note; `LevelAuditHandler.cpp:71-72` pre-seeds 100000 cm for this reason) and not in the wiki: `Docs/wiki-src/spatial.md:270` lists `probeLift` as a bare schema key with no default, units or purpose, and `:324` documents `overLandscape: false` only. Ask: register `GROUND_ABOVE_PROBE_START` in `Handlers/ErrorCodes.h` and emit it from the `bOverLandscape == true` branch with the probe-start Z, the landscape Z and `probeLift` named — registration in the same edit, because `PinWright.core.error_codes.AllEmittedCodesAreRegistered` (`Tests/Core/TestErrorCodeRegistry.cpp:460-462`) is a source scan that follows variable-forwarded codes (`:26-32`) and this one is forwarded via `Report.FailReasonCode`; `GROUND_ABOVE_PROBE_START` has zero matches in `Source/` today. Plus one `spatial.md` sentence on `probeLift`. Convention cited: `Docs/lessons.md:9` requires diagnostic context and an actionable suggestion in every caller-facing error — the sibling branch at `:717-722` honours it, this one does not. Dedup: board-wide search for `GROUND_NOT_FOUND`, `probeLift`, `probe_lift`, `overLandscape` returns zero files. Cross-linked `B-verify-grounding-maxgap-false-fail` (DONE, High) as the precedent — truthful numbers under an unhelpful verdict — and `B-ground-probe-hits-hull-not-render` (probe hits the wrong surface, not no surface). Severity Medium: the impact class is "soft blocker, doable only via a source dive" — the call refuses honestly and no wrong data is produced, so High's silent-false-success band does not apply, and it is above Low because the fix was not discoverable from the docs at all. Reach bump declined in both directions: `spatial.ground_*` is a placement-phase verb rather than an almost-every-session one, but the buried-start case is a normal path for anything scattered onto a plane before terrain, not a rare edge.
