---
id: E-volume-create-post-process-settings-no-echo
title: "volume.create_post_process_volume echoes its box config (priority/blendRadius/blendWeight/enabled/unbound) but never echoes the applied postProcessSettings/FPostProcessSettings overrides, so a caller can't confirm from the response whether the desaturation (bOverride_ColorSaturation/ColorSaturation) actually took"
status: OPEN
severity: Low
category: ergonomic
tags: [volume, create_post_process_volume, post-process, postprocesssettings, fpostprocesssettings, readback, echo, response-shape, no-echo, override, docs]
encounters: 1
lastSeen: 2026-07-01T19:06:56.5680937+03:00
---

# volume.create_post_process_volume accepts a postProcessSettings override map but echoes none of it, so a PPV placement with a color grade can't self-verify

`volume.create_post_process_volume` accepts an optional `postProcessSettings`
map — a partial `FPostProcessSettings` with `bOverride_*` flags and the paired
values (this task passed `{bOverride_ColorSaturation:true,
ColorSaturation:{0.5,0.5,0.5,1}}` for a desaturated look) — and applies it to the
spawned `APostProcessVolume->Settings`. Its success response echoes the box
config it applied (`priority`, `blendRadius`, `blendWeight`, `enabled`,
`unbound`) but not a single field of the `FPostProcessSettings` it just wrote.
So when a task's whole point is a specific grade ("a moody, desaturated look"),
the placement call gives the caller no in-band way to confirm the override
took — the response is identical whether `ColorSaturation` blended or was
silently dropped as an unrecognized key.

Because the placement doesn't echo which overrides applied, the caller cannot
tell a real desaturation from a no-op without a follow-up read, and the natural
whole-struct read-back spills: `actor.describe {PostProcessVolume, fields:
["properties"]}` returns the entire `FPostProcessSettings` (~57KB > the 10000-char
inline threshold) to a `Saved/PinWright/HttpResponses/.../<uuid>.json` file,
forcing a Grep + Read to extract the handful of `ColorSaturation`/`bOverride_*`
leaves that matter (the exact spill documented in `E-post-process-setters-no-echo`).

This is the same "the mutator already holds the answer but doesn't carry it, so a
verify forces extra reads / a spill gets paid" response-shape family as
`E-volume-add-post-process-echo` (OPEN, the sibling `volume.add_post_process_volume`
attach verb — echoes only `priority`, misses the box config), `E-post-process-setters-no-echo`
(OPEN, the `post_process.set_*` typed setters), `E-lighting-set-ao-exposure-no-echo`
(IN-REVIEW), and `E-pcg-set-self-pruning-no-echo` (OPEN). Per that family's stated
convention ("filed per-method because the fix is per-handler-response, not a
shared util"), this is the missing member for the `postProcessSettings` echo on
`volume.create_post_process_volume` — a distinct method + distinct field group
(the `FPostProcessSettings` override map, NOT the box `priority`/blend/enabled/
unbound config the create verb does already echo) that none of those tickets
covers.

## What it should do

Echo which overrides applied, only for keys the call actually supplied (they are
optional "configure the PPV" params) — the cheapest fix that lets the placement
self-verify. Mirror the `appliedSettings`/`overridesSet` shape
`set_volume_properties` already uses (its `propertiesSet` array), e.g. an
`overridesSet: ["ColorSaturation"]` list (or the resulting `FPostProcessSettings`
subset) built from the just-applied `PPV->Settings.*` values in hand at
response-build time — a handful of `SetStringField`/array writes on the `Resp`
object the handler already builds. Include the paired `bOverride_*` state (the
load-bearing thing a verifier wants — see
`E-property-get-override-state-ignores-boverride-bit` for why the value alone can
mislead). This removes the `actor.describe fields:["properties"]` spill entirely.

**Workaround:** to confirm a `create_post_process_volume` grade today, use
per-field `property.get { propertyName: "Settings.<Field>" }` (e.g.
`Settings.ColorSaturation`, and the paired `Settings.bOverride_ColorSaturation`
leaf) on the returned `volumeName` — each stays inline — instead of
`actor.describe fields:["properties"]`, which dumps the whole `FPostProcessSettings`
(~57KB) and spills to a HttpResponses file.

## Docs angle

`docs/wiki-src/volume.md` overlay should note that
`volume.create_post_process_volume` echoes only the box config
(priority/blendRadius/blendWeight/enabled/unbound), not the applied
`postProcessSettings` overrides, and name the per-field `property.get
Settings.<Field>` inline confirmation path (warning that
`actor.describe fields:["properties"]` on a PPV spills the whole ~57KB struct) —
the same overlay `E-volume-add-post-process-echo`, `E-volume-set-extent-units-class-dependent-docs`,
`E-volume-get-info-no-limit-spills`, `E-volume-type-filter-discovery`, and
`E-volume-create-name-vs-volumename` already target.

## Evidence

Struggle audit of the underground-arena gameplay/atmosphere-volumes task
(no seed, `volume` namespace, 13 executing RPCs + 2 wiki-nav doc calls, outcome
tool_bug — the judge filed `B-blocking-volume-no-brush-geometry` `#8` for the
TriggerBox extent defect; this is the separate PROCESS angle on the PPV placement).
The story's grade ask was explicit: "a post-process volume covering it for a
moody, desaturated look." The placement call `volume.create_post_process_volume
{Arena_PostProcess, location:{5000,0,-2000}, postProcessSettings:{bOverride_ColorSaturation:true,
ColorSaturation:{0.5,0.5,0.5,1}}}` (trace line 275) succeeded but echoed only
`priority`/`blendRadius`/`blendWeight`/`enabled`/`unbound` — never the applied
`FPostProcessSettings`. Friction note verbatim: *"volume.create_post_process_volume
accepted the postProcessSettings desaturation without error but never echoes the
applied FPostProcessSettings back, so I couldn't confirm the ColorSaturation
override actually took from the response alone."* No call errored — pure
verify overhead an echo-on-place would remove; the CallAnalyzer flagged it as a
distinct type-E "surprising" finding.

## Not a duplicate of

- `E-volume-add-post-process-echo` (OPEN) — the sibling `volume.add_post_process_volume`
  attach-to-actor verb, which echoes only `priority` and misses the box config
  (extent/blendRadius/blendWeight/enabled/unbound). This ticket is the standalone
  `create_post_process_volume` verb, which DOES echo that box config but misses the
  `postProcessSettings`/`FPostProcessSettings` override map. Different method,
  different response builder, different missing field group; complementary per the
  family's per-method convention.
- `E-post-process-setters-no-echo` (OPEN) — same no-echo family but a different
  method group (the `post_process.set_bloom/set_color_grading/...` typed setters in
  the `post_process` namespace); it does not touch the `volume.*` placement verbs.
- `E-lighting-set-ao-exposure-no-echo` (IN-REVIEW) — the `lighting.set_*` PPV
  setters; different method group.
- `E-property-get-override-state-ignores-boverride-bit` (OPEN) — that is the
  `property.get` `isOverridden` field reporting the wrong value once you read
  back; this is that the create verb doesn't echo the override at all, so you
  must read back, and the whole-struct read-back spills.

severity rationale: impact=response-echo gap recoverable via `property.get` / `actor.describe`, only forces extra reads (Low) x reach=rare (create_post_process_volume with a custom postProcessSettings grade is a specific atmosphere-setup path, not an every-session method) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the underground-arena gameplay/atmosphere-volumes task (no seed, `volume` namespace, 13 executing RPCs, outcome tool_bug — judge filed `B-blocking-volume-no-brush-geometry` `#8` for the TriggerBox extent defect; this is the separate PROCESS angle, and the CallAnalyzer flagged it as a distinct type-E "surprising" finding). `volume.create_post_process_volume {Arena_PostProcess, location:{5000,0,-2000}, postProcessSettings:{bOverride_ColorSaturation:true, ColorSaturation:{0.5,0.5,0.5,1}}}` (trace line 275) applied the desaturation override but its success response echoed only the box config `priority`/`blendRadius`/`blendWeight`/`enabled`/`unbound` — never the applied `FPostProcessSettings` — so the story's "moody, desaturated look" grade could not be confirmed from the placement response, and the whole-struct read-back (`actor.describe fields:["properties"]`) spills ~57KB to a HttpResponses file (per `E-post-process-setters-no-echo`). Friction note verbatim: *"volume.create_post_process_volume accepted the postProcessSettings desaturation without error but never echoes the applied FPostProcessSettings back, so I couldn't confirm the ColorSaturation override actually took from the response alone."* No call errored — pure verify overhead. Fix: echo an `overridesSet`/`appliedSettings` list (+ paired `bOverride_*` state) of the just-applied `PPV->Settings.*` keys in the placement `Resp` (cheap field writes), mirroring `set_volume_properties`'s `propertiesSet`; plus a `docs/wiki-src/volume.md` note naming the per-field `property.get Settings.<Field>` path. Same no-echo family as `E-volume-add-post-process-echo`/`E-post-process-setters-no-echo`/`E-lighting-set-ao-exposure-no-echo` (filed per-method). Deduped via ripgrep across OPEN/closed (create_post_process / postProcessSettings / FPostProcessSettings / overridesSet): no existing ticket scopes the `create_post_process_volume` postProcessSettings echo — `E-volume-add-post-process-echo` covers only the sibling `add_` attach verb's box config, not this verb's override map. Severity Low (recoverable via `property.get`; only forces reads; rare atmosphere-setup path).
