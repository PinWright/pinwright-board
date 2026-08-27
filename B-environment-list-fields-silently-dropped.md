---
id: B-environment-list-fields-silently-dropped
title: "environment list verb drops unknown fields[] keys and returns empty rows, the same defect just fixed on actor.list"
status: OPEN
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
