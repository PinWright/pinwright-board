---
id: F-gameplay-tags-add-no-batch
title: "No bulk gameplay_tags.add — seeding a tag taxonomy costs one RPC per tag into the same INI source"
status: OPEN
severity: Low
category: feature
tags: [no-batch-authoring, gameplay-tags, add, add_source, batch, ergonomic]
encounters: 1
lastSeen: 2026-07-11T14:19:17.7671264+03:00
---

# No bulk tag-registration primitive on `gameplay_tags`

## What's missing
Registering a gameplay-tag taxonomy is inherently a bulk operation, but there is
no one-shot primitive. `gameplay_tags.add` takes a single scalar `tag` (+ optional
`source`/`comment`), and `gameplay_tags.add_source` seeds **no** tags — it only
creates the empty INI source. So standing up an N-tag taxonomy costs N separate
`gameplay_tags.add` round-trips, every one writing the same INI file.

## What it should do
A batch capability that registers many tags into one source in a single RPC and
one atomic INI mutation. Either of:
- `gameplay_tags.add` accepting a `tags[]` array of `{tag, comment}` (alongside the
  existing single-tag form), or
- `gameplay_tags.add_source` growing an optional initial `tags[]` seed list so the
  source is created and populated in one call.

Returning one compact per-spec confirmation (`{tag, source, alreadyExisted}[]`).
This mirrors the existing per-namespace batch-convenience tickets
(`F-add-variables-batch`, `F-add-mapping-batch-keys`, `F-actor-batch-tag-set`,
`F-console-batch-get-cvar-values`, `F-animation-set-curve-key-no-batch`) — same
`no-batch-authoring` family (one-item-per-RPC authoring where a set-aware verb
should exist), a distinct method/fix surface each.

## Evidence (this task, outcome `clean` — no struggle, batch-capability gap only)
Task: stand up a combat-tag taxonomy in a dedicated `CombatTags.ini` source.
The agent created the source (`gameplay_tags.add_source source=CombatTags`), then
issued **6 separate `gameplay_tags.add` calls** into that same source, one dotted
tag per call — `Ability.Attack.Melee`, `Ability.Attack.Ranged`,
`Ability.Attak.Heavy` (a deliberate misspelling for the task), `Ability.Defense.Block`,
`State.Cooldown`, `State.Stunned` — plus a 7th corrective add after removing the
misspelled one (`Ability.Attack.Heavy`). All succeeded first try. The agent's own
narration flags why it serialized them: "I'll add them one at a time to avoid
concurrent writes to the same INI file." A batch add (or a seeded `add_source`)
would have collapsed the initial 6 single-tag writes into one atomic INI mutation
and removed the concurrency concern the agent raised. No error, retry, wrong
result, or `python.execute` fallback occurred — this is a convenience/ergonomic
gap, not a bug. The Judge disposition was "clean signal — no replay", confirming
every call behaves correctly.

severity rationale: impact=soft-workaround (N clean per-tag calls, here 6-for-1) x reach=occasional taxonomy-bootstrap path -> Low

## Distinct from
- `F-gameplay-tags-namespace` (DONE) — delivered the single-tag `add`/`remove`/`list`/`add_source`
  registry surface; it never scoped a bulk/seed form, so this batch gap is a follow-on, not a regression.
- `F-actor-batch-tag-set` (OPEN) — batch tagging of **actor** instances (`actor.add_tag`),
  a different namespace and fix surface from the project tag **registry**.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean gameplay-tag taxonomy task (namespace `gameplay_tags`, outcome `clean`, 13 RPC calls, Judge disposition "clean signal — no replay"). CallAnalyzer flagged the `workaround` pattern: 6 (then a 7th corrective) single-tag `gameplay_tags.add` RPCs into the same `CombatTags.ini`, with the agent narrating "I'll add them one at a time to avoid concurrent writes to the same INI file." `add_source` seeds no tags and `add` takes a single scalar, so N single-tag INI writes is the natural shape only because a batch does not exist. Proposed: a `tags[]` batch form on `gameplay_tags.add` (or a seeded `add_source`) collapsing N writes into one atomic mutation. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — `F-gameplay-tags-namespace` (DONE) covers only single-tag CRUD; `F-actor-batch-tag-set` is actor-instance tagging, a different namespace; no gameplay-tag-registry batch ticket exists. Genuinely new; seeded the `no-batch-authoring` family tag shared with `F-animation-set-curve-key-no-batch` / `F-add-mapping-batch-keys` / `F-add-variables-batch` / `F-actor-batch-tag-set` (per-method batch tickets that coexist and cross-reference).
