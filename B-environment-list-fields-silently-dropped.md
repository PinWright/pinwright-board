---
id: B-environment-list-fields-silently-dropped
title: "environment list verb drops unknown fields[] keys and returns empty rows, the same defect just fixed on actor.list"
status: IN-REVIEW
severity: High
category: bug
tags: [environment, fields, field-projection, silent-drop, allow-list, ReadFieldProjection]
encounters: 1
lastSeen: 2026-08-27
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
