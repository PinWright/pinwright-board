---
id: B-name-match-filter-ascii-only-case-fold
title: "NameMatch case-insensitive filters fold ASCII only, so a non-Latin label filter misses on actor.list but matches on drive.observe"
status: OPEN
severity: Low
category: bug
tags: [filter, name-match, case, unicode, sibling-divergence]
encounters: 1
lastSeen: 2026-10-03T00:00:00Z
rice: [1, 2, 1, 2]
priority: 8
---

# NameMatch case-insensitive filters fold ASCII only; drive.observe's label filter folds Unicode

`NameMatch::FFilter::Matches` (`Source/PinWright/Private/Utils/NameMatchFilter.cpp:72-110`)
implements the default case-insensitive path with `ESearchCase::IgnoreCase`. In UE 5.8 that path
ends in `TChar::ToUpper` / `ToLower` (`Engine/Source/Runtime/Core/Public/Misc/Char.h:77-91`), which
"Only converts ASCII characters". So `actor.list {filter:"трасса"}` does not match an actor labelled
`Трасса`, and the result is an empty list that reads as "no such actors". The same applies to
`system.inspect.list_objects`, `system.inspect.find_objects_by_class` and `spatial.raycast`
`actorFilter`.

`drive.observe` `label_contains` / `filter` / `handle_contains`
(`E-drive-observe-no-label-filter #4`) lower-case both sides through ICU
(`FText::ToLower`) on purpose, so `создать` matches `СОЗДАТЬ`. The `filter` key now has
two meanings for non-ASCII text, and neither wiki page says which one applies. rpc-design §21
("two verbs sharing a concept must share its semantics"; `NameMatch::FFilter` exists so this is
"one decision, not one per verb") calls that divergence a defect.

## What it should do

Converge on one fold. Do the non-ASCII-safe lower-casing in one place, e.g.
`FTextTransformer::ToLower` on both sides inside `FFilter::Matches` when `!bCaseSensitive` for
contains, prefix and exact. Regex mode would need ICU's `UREGEX_CASE_INSENSITIVE`. Then have
`drive.observe` reuse `NameMatch::FFilter`. This is a behaviour change for the NameMatch consumers,
because non-ASCII patterns would match more rows, so it needs a CHANGELOG entry and a test with a
Cyrillic label on `actor.list`.

## History
- `#1-initial-report` `OPEN` reviewer - Found while reviewing `E-drive-observe-no-label-filter #4`, at plugin `7230b41d` plus that working-tree change. `NameMatch::FFilter::Matches` uses `ESearchCase::IgnoreCase`, which is ASCII-only in UE 5.8 (`Char.h:77-91`). The new `drive.observe` filters fold through ICU. The same `filter` key therefore behaves differently for Cyrillic labels on `actor.list` than on `drive.observe`. No live repro was run; this follows from reading the source. Dedup: the board has `B-actor-list-filter-case-mismatch` (DONE; about ASCII case sensitivity, not non-Latin folding) and `B-asset-list-class-filter-case-divergence`. No existing ticket covers non-ASCII folding.
