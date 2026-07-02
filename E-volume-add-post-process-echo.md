---
id: E-volume-add-post-process-echo
title: "volume.add_post_process_volume applies extent/priority/blendRadius/blendWeight/enabled/unbound but echoes only priority — the placement can't self-verify, forcing a get_volumes_info + per-field property.get round-trip to confirm the config took"
status: OPEN
severity: Low
category: ergonomic
tags: [volume, add_post_process_volume, post-process, readback, echo, response-shape, round-trip, no-echo, docs]
encounters: 1
lastSeen: 2026-07-01T16:20:24.1200373+03:00
---

# `volume.add_post_process_volume` echoes only `priority`, none of the other config it applied

`volume.add_post_process_volume` accepts and applies a full post-process-volume
config — `extent`, `priority`, `blendRadius`, `blendWeight`, `enabled`,
`unbound` (`bUnbound`) — but its success response carries **only `priority`**
among them. The return object is `{volumeName, volumeClass, attachedTo,
priority, attachmentSucceeded, actorPath}` (plus the AddActor verification block)
— `extent`/`blendRadius`/`blendWeight`/`enabled`/`unbound` are **set-but-not-echoed**,
even though the handler wrote all of them onto the spawned `APostProcessVolume`.

Because the placement doesn't echo what it applied, a "make sure it landed with
the settings you asked for" verify (exactly this task's closing ask) has **no
in-band answer from the placement call**. The natural inventory verb
`volume.get_volumes_info` doesn't close the gap either — it emits only
name/class/location/extent per row (the separate
`E-volume-get-info-omits-physics-properties` gap), so the config fields are absent
there too. With neither the placement return nor the info readback surfacing
priority/blend/enabled/unbound, the only confirmation path left is **N separate
`property.get` calls** (this task fired 5: `Priority`, `BlendRadius`,
`BlendWeight`, `bEnabled`, `bUnbound`).

This is the same "the mutator already holds the answer but doesn't carry it, so a
verify forces extra reads" response-shape family as
`E-post-process-setters-no-echo` (OPEN, the `post_process.set_*` typed setters),
`E-lighting-set-ao-exposure-no-echo` (IN-REVIEW), `E-pcg-set-self-pruning-no-echo`
(OPEN), and their siblings. Per that family's stated convention ("filed per-method
because the fix is per-handler-response, not a shared util"), this is the missing
member for **`volume.add_post_process_volume`** — a distinct method (namespace
`volume`, the placement verb) that none of those tickets covers.

## What it should do

Echo the full applied config in the placement response, only for params the call
actually applied (they are optional "configure the PPV" params) — a handful of
`SetNumberField`/`SetBoolField` writes on the `Resp` object the handler already
builds, with the values in hand at response-build time from the just-set
`PPV->Settings`/`PPV->` fields:

- `extent` (the box extent it sized to), `blendRadius`, `blendWeight`, `enabled`
  (`bEnabled`), `unbound` (`bUnbound`) — alongside the `priority` it already
  echoes.

This makes the placement **self-verifying**: a single call confirms the config
took, removing the follow-up `get_volumes_info` + 5×`property.get` round-trip.
It composes with (does not duplicate) `E-volume-get-info-omits-physics-properties`
— that ticket adds a `postProcessProperties` block to the *inventory* readback;
this adds the echo to the *placement* return. Either alone closes this task's
verify; together they give both a self-verifying placement and a confirmable
inventory scan (the same both-fixes-compose stance `E-post-process-setters-no-echo`
takes for the setter/verify loop).

**Workaround:** confirm a placement today with per-field
`property.get { propertyName: "Priority" | "BlendRadius" | "BlendWeight" |
"bEnabled" | "bUnbound" }` on the returned `volumeName` (each stays inline), plus
`get_volumes_info` for location/extent.

## Docs angle

`docs/wiki-src/volume.md` overlay should note that `volume.add_post_process_volume`
echoes only `priority` (not extent/blend/enabled/unbound) and point at the
per-field `property.get` path for confirmation until the echo lands — the same
overlay `E-volume-get-info-omits-physics-properties`, `E-volume-get-info-no-limit-spills`,
`E-volume-type-filter-discovery`, `E-volume-create-name-vs-volumename`, and
`E-volume-set-extent-units-class-dependent-docs` already target.

## Evidence

Struggle audit of the "anchor a bounded post-process volume on a landmark" task
(focus `volume.add_post_process_volume`, namespace `volume`, 15 MCP calls,
outcome clean/done — the judge filed nothing on this method because the placement
succeeded; this is the PROCESS angle). The story's closing ask was explicitly a
readback: "make sure it actually landed at the right spot with the settings you
asked for" (priority/blendRadius/blendWeight/enabled/unbound). The placement call
`volume.add_post_process_volume {UELogo, extent700, pri10, blendR250, blendW1,
enabled, unbound=false}` applied all six params, but its return echoed only
priority: `{"volumeName":"UELogo_PostProcessVolume","volumeClass":"APostProcessVolume",
"attachedTo":"UELogo","priority":10,"attachmentSucceeded":true,"actorPath":...}`
— no extent/blendRadius/blendWeight/enabled/unbound. Friction note (verbatim):
*"add_post_process_volume also only echoes priority (not blend/enabled/unbound)."*
The verify therefore fell to 5 separate `property.get` calls (`Priority=10`,
`BlendRadius=250`, `BlendWeight=1`, `bEnabled=true`, `bUnbound=false`) after
placement. No call errored — pure verification overhead an echo-on-place would
remove. Source (per the sibling ticket's HEAD replay): the handler SETs
`Priority`/`BlendRadius`/`BlendWeight`/`bEnabled`/`bUnbound` (`VolumeHandler.cpp:1998-2002`)
but its `Resp` builder writes only `volumeName`/`volumeClass`/`attachedTo`/`priority`/
`attachmentSucceeded` + AddActor verification (`:2006-2012`, priority at `:2010`).

## Not a duplicate of

- `E-volume-get-info-omits-physics-properties` (OPEN, the judge's filing this task
  appended to) — that is the *inventory readback* gap (`get_volumes_info` omits the
  config for PhysicsVolume/PostProcessVolume rows); this is the *placement return*
  echo gap on `add_post_process_volume`. Different method, different handler
  response; complementary fixes. That ticket mentions this echo gap only as context
  for why the agent fell back to `get_volumes_info` — it does not scope its fix to
  the placement verb.
- `E-post-process-setters-no-echo` (OPEN) — same no-echo family but a different
  method group (the `post_process.set_bloom/set_color_grading/...` typed setters in
  the `post_process` namespace); it does not touch `volume.add_post_process_volume`.
  Filed per-method per that ticket's own convention.
- `F-post-process-typed-setters` (DONE) — added the `post_process.set_*` setters; a
  different surface from the `volume.*` placement verb.

severity rationale: impact=response-echo gap recoverable via `property.get`, only forces extra reads (Low) × reach=rare (add_post_process_volume is a specific PPV-placement path, not an every-session method) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the "anchor a bounded post-process volume on a landmark" task (focus `volume.add_post_process_volume`, namespace `volume`, 15 MCP calls, outcome clean/done — judge filed nothing on this method; this is the separate PROCESS angle, and the CallAnalyzer flagged it as a distinct type-E "surprising" finding). `volume.add_post_process_volume` applied all six config params (extent/priority/blendRadius/blendWeight/enabled/unbound) but echoed only `priority`: return `{volumeName, volumeClass:"APostProcessVolume", attachedTo:"UELogo", priority:10, attachmentSucceeded:true, actorPath}` (source `VolumeHandler.cpp:2006-2012`, priority `:2010`; sets at `:1998-2002`). With neither the placement echo nor `get_volumes_info` (see `E-volume-get-info-omits-physics-properties`) surfacing blend/enabled/unbound, the story's closing "confirm it landed with the settings you asked for" verify forced 5 separate `property.get` calls (`Priority`/`BlendRadius`/`BlendWeight`/`bEnabled`/`bUnbound`). Friction note verbatim: *"add_post_process_volume also only echoes priority (not blend/enabled/unbound)."* No call errored — pure verify overhead. Fix: echo the applied config (extent/blendRadius/blendWeight/enabled/unbound) in the placement `Resp` (cheap field writes off the just-set `PPV->` values), making the placement self-verifying; plus a `docs/wiki-src/volume.md` note. Same no-echo family as `E-post-process-setters-no-echo`/`E-lighting-set-ao-exposure-no-echo` (filed per-method). Deduped via ripgrep across OPEN/closed (add_post_process_volume / echo / no-echo): no existing ticket scopes the placement-verb echo — `E-volume-get-info-omits-physics-properties` covers only the inventory readback and mentions this echo gap merely as context. Severity Low (recoverable via `property.get`; rare PPV-placement path).
