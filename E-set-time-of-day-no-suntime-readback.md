---
id: E-set-time-of-day-no-suntime-readback
title: "environment.build.set_time_of_day neither echoes the resulting value nor documents that the modern sky sphere stores time-of-day as the 'Sun height' double — verifying the TOD costs two PROPERTY_NOT_FOUND guesses"
status: OPEN
severity: Low
category: ergonomic
tags: [environment, set-time-of-day, sun-height, readback, property-get, time-of-day, sky-sphere, discoverability, docs]
encounters: 3
lastSeen: 2026-07-13T08:23:09Z
---

# `set_time_of_day` gives no readback and no hint that the modern sky sphere's time slot is `Sun height`

`environment.build.set_time_of_day {time:18.5}` succeeds, but a caller who then
wants to **verify the value took** (read the time-of-day back off the sky sphere)
has no signposted path:

1. The handler does **not echo the value it set**. On the modern
   `/Engine/EngineSky/BP_Sky_Sphere` it maps `time` (0..24) onto the actor's
   `Sun height` double via `SunHeight = -cos((time/24)*2pi)`
   (`EnvironmentHandler.cpp:400-432`) but the response carries only
   `{action:"set_time_of_day"}` — neither the requested `time` nor the computed
   `Sun height`. So the round-trip "set, then confirm" forces a separate
   `property.get`.
2. There is **no documentation of which property to read**. The intuitive names
   for a time-of-day control — `Time of Day`, `TimeOfDay` — do **not** exist on
   `BP_Sky_Sphere_C`; the modern sphere's TOD control is the double literally
   named **`Sun height`** (two words, with a space). Nothing in the wiki tells the
   caller this: `docs/wiki-src/environment.md` has zero mention of
   `set_time_of_day` or `Sun height`, and the auto-generated
   `environment.build.set_time_of_day` method page documents only the `time`
   input, not the readback property. The only place `Sun height` is written down
   is the C++ handler and the `B-create-sky-sphere-stale-path #2` developer note
   — neither reachable by an agent at call time.

So the natural "did 18.5 take?" verification is pure trial-and-error: two
`property.get` guesses both hard-fail `PROPERTY_NOT_FOUND` before the agent
resorts to reading the plugin source to learn the property name.

## Repro (verbatim, from the audited evening-blockout greybox task)

Namespace `environment.build`; outcome `nonrepro` (judge tracked the unrelated
`export_snapshot` stub under `B-export-snapshot-empty-stub`). The TOD-verify
friction:

1. `environment.build.set_time_of_day {time:18.5}` → success (response:
   `{action:"set_time_of_day"}`, no value echoed).
2. `property.get {SkySphere, "Time of Day"}`
   → `[PROPERTY_NOT_FOUND] Property Time of Day not found on object
   /Game/Maps/ExampleProjectWelcome...BP_Sky_Sphere_C_0.`
3. `property.get {SkySphere, "TimeOfDay"}`
   → `[PROPERTY_NOT_FOUND] Property TimeOfDay not found on ...BP_Sky_Sphere_C_0.`
4. (after reading the handler source) `property.get {SkySphere, "Sun height"}`
   → success, `Sun height = -0.13053` = `-cos((18.5/24)*2pi)`, confirming the set.

Friction note (verbatim): *"BP_Sky_Sphere exposes no literal TimeOfDay UPROPERTY
so two property.get guesses ('Time of Day', 'TimeOfDay') hit PROPERTY_NOT_FOUND
before I read the plugin handler and learned TOD is stored as 'Sun height'."*

Two wasted `is_error` calls, corrected only by reading source — pure
discoverability/round-trip overhead, no blocked progress.

## What it should do

Two cleanly-separable, low-disruption improvements (either alone removes most of
the friction):

1. **Echo the value on the response (ergonomic).** Have
   `environment.build.set_time_of_day` add the requested `time` and the computed
   `Sun height` (and/or whichever variable it actually wrote) to the response
   object, so the set is self-verifying and no follow-up `property.get` is needed.
   The handler already holds both numbers at `EnvironmentHandler.cpp:415`; this is
   a couple of `SetNumberField` calls next to the existing
   `SetStringField("action", ...)` at `:432`.
2. **Document the readback property (docs).** Tag `docs`, edit
   `docs/wiki-src/environment.md`: add a short note that
   `environment.build.set_time_of_day` drives the modern
   `/Engine/EngineSky/BP_Sky_Sphere`'s **`Sun height`** double (mapping
   `time → -cos((time/24)*2pi)`; midnight → -1, noon → +1), and that there is no
   `TimeOfDay`/`Time of Day` UPROPERTY — so a verify reads `property.get` on
   `Sun height`, not the intuitive name. (The wiki edit itself is the downstream
   process, not this ticket.)

## Not a duplicate of

- `B-create-sky-sphere-stale-path` (IN-REVIEW) — that fixes the dead
  `create_sky_sphere` load path and aligns `set_time_of_day`'s sky-sphere
  *discovery* so the set works at all; its `#2` dev note mentions `Sun height` as
  an implementation detail. It does **not** add a value echo to the response nor a
  caller-facing doc/readback signpost — the PROCESS gap here is "the set worked,
  but verifying it is undiscoverable," which survives that fix.
- `B-export-snapshot-empty-stub` (IN-REVIEW; judge-tracked this task) — the
  `export_snapshot` stub OUTCOME bug; orthogonal.
- `E-environment-build-create-no-name-param` (OPEN) — naming the *created* sky/fog
  actors; this is the time-of-day *readback*, a different verb and intent.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle (process) audit of the
  evening-blockout greybox task (namespace `environment.build`; outcome
  `nonrepro`; judge tracked `B-export-snapshot-empty-stub`). Distinct PROCESS
  angle: verifying that `set_time_of_day {time:18.5}` took cost two
  `PROPERTY_NOT_FOUND` `property.get` guesses (`"Time of Day"`, `"TimeOfDay"`)
  before the agent read the handler source and learned the modern
  `BP_Sky_Sphere`'s TOD slot is the `Sun height` double (`property.get` then
  returned `-0.13053 = -cos((18.5/24)*2pi)`). Two root causes: (a)
  `set_time_of_day` echoes only `{action}` and not the value it set
  (`EnvironmentHandler.cpp:415` computes `Sun height` but `:432` doesn't emit it),
  so the set isn't self-verifying; (b) the readback property name is undocumented
  — `docs/wiki-src/environment.md` never mentions `set_time_of_day` or
  `Sun height`, and the auto method page lists only the `time` input. Dedup
  (ripgrep over OPEN+closed; qmd unavailable): not covered by
  `B-create-sky-sphere-stale-path` (create/set fix, not readback discoverability),
  `B-export-snapshot-empty-stub`, or `E-environment-build-create-no-name-param`.
  Fix: echo `time`/`Sun height` on the `set_time_of_day` response, and/or a
  `docs/wiki-src/environment.md` note naming `Sun height` as the readback slot.
- `#2-additional-golden-hour` `OPEN` reporter — Additional evidence (golden-hour
  cinematic-lighting task, seed `environment.build.create_sky_sphere`, replay-confirmed
  against `mcp__editor-automation__call`): the same no-echo / wrong-property-name friction
  reproduces with `time=17.5` (positive `Sun height`, vs the `#1` task's `18.5` →
  negative). Verbatim replay: `environment.build.set_time_of_day {time:17.5}` →
  `{"success":true,"action":"set_time_of_day"}` (value not echoed);
  `property.get {objectPath:.../BP_Sky_Sphere_C_0, propertyName:"Time of day"}` →
  `[PROPERTY_NOT_FOUND] Failed to resolve property 'Time of day' on object
  /Game/Maps/ExampleProjectWelcome...BP_Sky_Sphere_C_0: Property 'Time of day' not
  found` (a third intuitive spelling — space + lowercase — that also dead-ends, on
  top of `#1`'s `"Time of Day"`/`"TimeOfDay"`); the real slot
  `property.get {propertyName:"Sun height"}` → `value=0.13052606581920428`
  = `-cos((17.5/24)*2pi)`, matching `EnvironmentHandler.cpp:415`. Confirms the
  documented two root causes and widens the set of failing intuitive names. (Also
  noted in this task but NOT a separate file: `actor.find_by_class {className:"BP_Sky_Sphere"}`
  returns a clean, correct, guidance-bearing `[CLASS_NOT_FOUND]` — expected behavior
  for a short name that is an asset BP, not a `/Script` class.)
- `#3-liveness` `OPEN` reporter — still reproduces at HEAD (dusk-landscape blockout, `time=18.5`, same shape as `#1`): `set_time_of_day {time:18.5}` → `{"success":true,"action":"set_time_of_day"}` (no value echoed); `property.get {propertyName:"Sun height"}` → `-0.13052632584380217 = -cos((18.5/24)*2pi)`; `property.list {nameMatch:"time"}` returns only `CustomTimeDilation` + `RuntimeGrid` — confirming no 0..24 time-of-day UPROPERTY exists on `BP_Sky_Sphere_C`.
