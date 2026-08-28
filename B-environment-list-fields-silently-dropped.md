---
id: B-environment-list-fields-silently-dropped
title: "environment list verb drops unknown fields[] keys and returns empty rows, the same defect just fixed on actor.list"
status: DONE
severity: High
category: bug
tags: [environment, fields, field-projection, silent-drop, allow-list, ReadFieldProjection]
encounters: 2
lastSeen: 2026-08-28T10:05:00+05:00
---

# The same unvalidated `fields` allow-list, one file over

`Handlers/Environment/EnvironmentHandler.cpp:1060` declares a byte-identical four-key `fields`
allow-list to the one `actor.list` carried, and reads it through the same unvalidated
`ReadFieldProjection` helper. So the same two defects are present: a key outside the allow-list is
silently discarded rather than rejected, and a projection made only of unrecognised keys returns
`success` with rows that are empty JSON objects — which reads as "this actor has no such data"
rather than "the verb refused to answer".

Found while fixing `B-actor-list-fields-unknown-key-silently-dropped`, by the agent that fixed it.
It was out of that ticket's file scope and deliberately not widened into.

**Fix:** the same shape that landed on `actor.list` — validate the projection against the emittable
set immediately after `ReadFieldProjection` and reject an unknown entry with `INVALID_PARAMS` naming
the offending key and the valid set.

## History
- `#1-found-while-fixing-the-sibling` `OPEN` reporter — Found by the agent fixing
  `B-actor-list-fields-unknown-key-silently-dropped`, which rejects unknown keys and adds `folder` on
  `actor.list`. This verb was outside that ticket's ownership. Not reproduced live; the claim is a
  source reading of the allow-list and the shared helper.
- `#2-reject-unknown-fields-keys` `IN-REVIEW` developer — Re-read against current source (not
  run): the verb is `system.inspect.list_objects` (registered at
  `Handlers/Environment/EnvironmentHandler.cpp:1053`, `ReadFieldProjection` at :1082 — the file's
  only field projection) and the defect was present exactly as filed. Added an `EmittableFields`
  allow-list plus a validation pass immediately after `ReadFieldProjection` that rejects any key the
  row builder cannot emit with `INVALID_PARAMS`, naming the offending entries and the valid set; the
  `fields` param description now states the rejection. No key was ADDED: `folder` is already
  projectable on the `actor.list` twin and the wider per-actor keys on
  `system.inspect.inspect_object`, so the rejection message routes callers to those siblings instead
  of widening the rows of a verb that already spills. The code is spelled as a raw `TEXT("INVALID_PARAMS")`
  literal, matching the file's 40 other hand-spelled codes — a single `ErrorCodes::ERR_` reference
  would flip this non-baselined file to "adopting" and fail
  `PinWright.core.error_codes.RegistryAdoptingFilesUseConstantsOnly`. Test:
  `PinWright.system.inspect.list_objects.FieldProjection.UnknownFieldIsRejected` in
  `Tests/Environment/TestListObjectsFieldProjection.cpp`.

- `#3-verified-fixed` `DONE` verifier — 2026-08-28. Repro run live against the editor rebuilt at
  `b79ba53e`, on `/Game/Maps/Atlantis`, against the verb `#2` names (`system.inspect.list_objects`).
  Both grades of the defect are closed, and the good case is unaffected.
  **All-unknown grade** (the one that used to return success with `{}` rows):
  `system.inspect.list_objects {filter:"ExponentialHeightFog", fields:["folder"]}` ->
  `[INVALID_PARAMS] Unknown fields[] entry(s) for 'system.inspect.list_objects': [folder]. Valid
  fields: [label, name, path, class]. Rejected rather than dropped: a projection made only of
  unrecognised keys returns rows that are empty objects. For the Outliner folder use actor.list with
  fields=["folder"]; for properties, transform or components use system.inspect.inspect_object.`
  **Partial-drop grade:** `fields:["name","bogusKey"]` -> the same `INVALID_PARAMS`, naming
  `[boguskey]` and the same valid set — so a mixed projection no longer succeeds while silently
  losing a column. **Differential control:** `fields:["name","class"]` -> `success:true`,
  `{"name":"ExponentialHeightFog_0","class":"ExponentialHeightFog"}`, `count:1`, `totalMatches:1` —
  populated rows, no error, so the rejection is scoped to unrecognised keys and is not a blanket
  refusal of projection. This is a behaviour change, not a diagnostic one: the failing call used to
  return `success` with empty row objects and now returns no rows at all. The rejection message
  routes `folder` to `actor.list` rather than widening this verb, exactly as `#2` states; `actor.list
  {fields:["folder"]}` is separately verified on `B-actor-list-fields-unknown-key-silently-dropped`.
