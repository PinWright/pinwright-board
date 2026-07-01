---
id: E-post-process-setters-no-echo
title: "post_process.set_bloom/set_color_grading/set_lumen_gi/set_lumen_reflections/set_motion_blur echo only {success} — none of the FPostProcessSettings values they wrote — so confirming the grade forces an actor.describe fields=[properties] that dumps the whole 57KB struct and spills to a HttpResponses file"
status: OPEN
severity: Low
category: ergonomic
tags: [post-process, set-bloom, set-color-grading, set-lumen-gi, set-lumen-reflections, set-motion-blur, readback, round-trip, response-shape, echo, response-size, oversized, spill, docs]
encounters: 1
lastSeen: 2026-07-01T09:21:27.2934501+03:00
---

# The five non-AA `post_process.set_*` setters don't echo what they wrote, so a "confirm it took effect" read-back has no cheap path and the natural one spills 57KB

The typed PPV setters added by DONE `F-post-process-typed-setters` —
`post_process.set_bloom`, `set_color_grading`, `set_lumen_gi`,
`set_lumen_reflections`, `set_motion_blur` — each write several
`FPostProcessSettings` fields (and flip the paired `bOverride_*` flags) onto a
find-or-spawned PostProcessVolume, but their success responses carry **none of
the values they just applied**. (The sixth setter, `set_anti_aliasing`, is the
exception: it echoes `{success, antiAliasingMethod, screenPercentage}`, which is
why AA can be confirmed cheaply — see below.) When the task explicitly asks to
"confirm the settings took effect," the caller has no in-band answer from the
setter, and the natural whole-volume read-back overflows:
`actor.describe {actorName: PostProcessVolume_0, fields:["properties"]}` returns
`outputTooLong` (57228 chars > the 10000-char inline threshold) and spills the
entire `FPostProcessSettings` to a `Saved/PinWright/HttpResponses/.../<uuid>.json`
file, forcing a follow-up Grep + Read to extract the handful of fields that
matter. The `fields:["properties"]` projection does **not** help: the whole
struct lives under that one top-level `properties` key, so projecting to it still
returns the full 57KB.

This is the same "the mutator already holds the answer but doesn't carry it, so a
working readback verb gets spammed / a spill gets paid" response-shape family as
`E-lighting-set-ao-exposure-no-echo` (IN-REVIEW),
`E-set-transition-settings-no-echo` (OPEN),
`E-geometry-deformer-echo-mesh-counts` (OPEN),
`E-create-procedural-terrain-no-material-echo` (OPEN), and
`E-niagara-modify-parameter-no-override-readback` (OPEN). Per that family's
stated convention ("filed per-method because the fix is per-handler-response,
not a shared util"), this is the missing member for the **`post_process.set_*`**
setters — a distinct method group none of those tickets covers (the lighting
ticket scopes only to `lighting.set_ambient_occlusion` / `lighting.set_exposure`).

## Friction evidence (this task — focus `post_process.set_anti_aliasing`, namespace `post_process`, finalize-the-look task, outcome tool_bug)

The judge filed the outcome bug on `set_anti_aliasing`
(`B-set-aa-invalid-method-silent-noop`, Medium); this is the separate PROCESS
angle on the *other five* setters. The story's closing ask was a first-class
read-back: "confirm ... that Lumen GI and reflections really are enabled on the
level's post-process settings" (also the AA method + screen percentage). The
agent applied all six setters one-shot (`set_bloom intensity=0.4 threshold=-1`,
`set_color_grading whiteTemp=7000 contrast=1.1 saturation=1.1`, `set_lumen_gi
enabled=true finalGatherQuality=2`, `set_lumen_reflections enabled=true
quality=2`, `set_motion_blur amount=0.5 targetFps=60`) — none echoed the applied
values. To confirm:

- **AA** was verified cheaply via `system.console.search` (`r.AntiAliasingMethod`
  currentValue=4/TSR, `r.ScreenPercentage`=80) — because `set_anti_aliasing`
  echoes its method and CVar-backed values, so no spill was needed.
- **Bloom / color-grading / Lumen GI / Lumen reflections / motion-blur** had no
  echo and no cheap getter, so the agent fell back to
  `actor.describe {actorName: PostProcessVolume_0, fields:["properties"]}` (trace
  line 269) → `{"outputTooLong":true,"message":"Response exceeds display limit
  (57228 chars, threshold 10000); full payload written to ...Saved/PinWright/
  HttpResponses/.../...json"}`, then Grep'd the spill file for
  `GlobalIlluminationMethod|ReflectionMethod|Bloom|WhiteTemp|ColorContrast|
  ColorSaturation|MotionBlur` (line 272) and Read it at offset 300 (line 299) to
  see the `ColorSaturation`/`ColorContrast` arrays.

Friction note (verbatim): *"only minor overhead was actor.describe spilling its
57KB payload to a Saved/PinWright/HttpResponses file (documented behavior)
requiring a Grep/Read to read the values back."* Call log: 12 MCP calls (2
wiki-nav + 10 RPCs), all `success:true`, no retries; the `actor.describe` +
Grep + Read at the end are the recovery an echo-on-set would have removed. All
succeeded — pure PROCESS overhead; the seed outcome was the AA bug, not this.

## What it should do

Primary (matches the no-echo family, cheapest fix): have the five non-AA
`post_process.set_*` setters echo the values they applied in their success
responses, only for params present in the call (all are optional-param "update
settings" RPCs), plus an override/`enabled` indicator (the `bOverride_*`
foot-gun is the load-bearing thing a verifier wants):

- `set_bloom` → `intensity`, `threshold`, `method`.
- `set_color_grading` → `whiteTemp`, `tint`, `saturation`, `contrast`, `gamma`,
  `gain`.
- `set_lumen_gi` → `enabled` (the `DynamicGlobalIlluminationMethod` state) +
  `sceneDetail` / `finalGatherQuality` / `maxTraceDistance`.
- `set_lumen_reflections` → `enabled` (the `ReflectionMethod` state) +
  `quality` / `rayLightingMode` / `maxRoughnessToTrace`.
- `set_motion_blur` → `amount`, `max`, `targetFps`, `perObjectSize`.

Cheap: a few `SetNumberField`/`SetBoolField`/`SetStringField` calls on the `Resp`
object each handler already builds (values read off `PPV->Settings.*` in hand at
response-build time), removing the read-back round-trip and the spill entirely —
exactly the fix landed for `lighting.set_*` in `E-lighting-set-ao-exposure-no-echo`
(#3).

Alternative (the call-trace analyzer's original proposal, if a single verify verb
is preferred over per-setter echoes): a convenience `post_process.get` returning
only the currently-overridden PP fields in a compact inline payload — one call
for a multi-field verify instead of 5–8 `property.get` calls or the whole-struct
`actor.describe`. Or teach `actor.describe` to project into nested struct
sub-fields (so `fields:["properties.BloomIntensity", ...]` narrows) — but that is
a broader `actor.describe` change; the per-setter echo is the targeted one.

**Workaround:** to confirm a post_process write today, use per-field
`property.get { propertyName: "Settings.<Field>" }` (e.g. `Settings.BloomIntensity`,
`Settings.DynamicGlobalIlluminationMethod`, `Settings.ReflectionMethod`,
`Settings.ColorSaturation`, `Settings.MotionBlurAmount`) — each stays inline —
**instead of** `actor.describe fields:["properties"]`, which dumps the entire
`FPostProcessSettings` (~57KB) and spills to a HttpResponses file. Read the paired
`Settings.bOverride_<Field>` leaf to confirm the value actually blends (see
`E-property-get-override-state-ignores-boverride-bit` for why `isOverridden` can
mislead here).

## Docs angle

`docs/wiki-src/post_process.md` overlay should (a) note the five non-AA typed
setters don't echo their applied values, (b) name the compact read-back path
(`property.get Settings.<Field>` per field) and **warn** that
`actor.describe fields:["properties"]` on a PPV returns the whole
`FPostProcessSettings` (~57KB) and spills, and (c) cross-link
`E-property-get-override-state-ignores-boverride-bit` for the `isOverridden`
value-vs-default caveat on `bOverride_`-gated fields.

## Not a duplicate of

- `E-lighting-set-ao-exposure-no-echo` (IN-REVIEW) — same response-echo family
  and root cause, but a different method group (`lighting.set_ambient_occlusion`
  / `set_exposure`); it is scoped to those two and does not touch the six
  `post_process.set_*` setters. Filed per-method per that ticket's own stated
  convention.
- `B-set-aa-invalid-method-silent-noop` (OPEN, the judge's filing for this task)
  — the `set_anti_aliasing` *silent no-op on an invalid method token* ground-truth
  bug; this is the read-back/echo *response-shape* gap on the other five setters
  (AA already echoes and did not need the spill).
- `F-post-process-typed-setters` (DONE) — *added* these setters and explicitly
  listed "Reading current values back; `asset.dump` … covers that" as **out of
  scope, deferred until requested**. This task is that request; this ticket is the
  deferred read-back/echo gap, proposing the cheaper per-setter echo.
- `E-property-get-override-state-ignores-boverride-bit` (OPEN) — that is the
  `property.get` `isOverridden` field *reporting the wrong value* once you read
  back; this is that the setters don't echo *at all*, so you must read back, and
  the whole-struct read-back *spills*.
- `E-volume-get-info-no-limit-spills` (OPEN) — same spill-to-HttpResponses tax
  but on a different method (`volume.get_volumes_info` dumping every level
  volume); this is the `post_process` setter/verify loop specifically.

severity rationale: impact=response-spill that only forces a Read (Low) × reach=rare (post_process grading is a specific finalize-the-look setup path, not an every-session method) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the finalize-the-look `post_process` task (focus `post_process.set_anti_aliasing`, 12 calls, outcome tool_bug — judge filed `B-set-aa-invalid-method-silent-noop` for the AA silent-noop; this is the separate PROCESS angle). The five non-AA typed setters (`set_bloom`/`set_color_grading`/`set_lumen_gi`/`set_lumen_reflections`/`set_motion_blur`) echo only `{success}` (+ actor verification), none of the `FPostProcessSettings` values they wrote, while the user's closing ask was explicitly to "confirm ... Lumen GI and reflections really are enabled." With no echo and no compact getter, the agent read back via `actor.describe {PostProcessVolume_0, fields:["properties"]}` (trace line 269), which overflowed (`outputTooLong`, 57228 chars > 10000 threshold), spilled to `Saved/PinWright/HttpResponses/.../<uuid>.json`, and forced a Grep (line 272) + Read at offset 300 (line 299); `fields:["properties"]` did not narrow it because the whole struct sits under one `properties` key. Friction note (verbatim above): *"only minor overhead was actor.describe spilling its 57KB payload to a Saved/PinWright/HttpResponses file ... requiring a Grep/Read to read the values back."* AA itself was confirmed cheaply via `system.console.search` because `set_anti_aliasing` DOES echo — underscoring that the gap is the five non-echoing setters. Same "mutator should carry the answer" shape as `E-lighting-set-ao-exposure-no-echo` (IN-REVIEW) and its siblings; the read-back that `F-post-process-typed-setters` (DONE) deferred "until requested" — now requested. Fix: echo the applied values (+ override/`enabled` indicator) in each setter's response (cheap `Resp` field writes off `PPV->Settings.*`); alternatively a compact `post_process.get`. Docs: `docs/wiki-src/post_process.md` should warn that `actor.describe fields:["properties"]` on a PPV spills the whole 57KB struct and name the per-field `property.get Settings.<Field>` inline path. Deduped via ripgrep across OPEN/closed (post_process / no-echo / readback / echo): no existing ticket covers the `post_process.set_*` response-echo angle. Severity Low (recoverable via `property.get`; only forces a Read; rare finalize-the-look path).
