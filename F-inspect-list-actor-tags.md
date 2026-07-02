---
id: F-inspect-list-actor-tags
title: "No way to enumerate the distinct actor Tags present in a world — find_by_tag requires an already-known exact tag, so a 'is anything tagged for review/cleanup' audit can only guess"
status: IN-REVIEW
severity: Medium
category: feature
tags: [inspection, find_by_tag, actor-tags, enumeration, discovery, audit, tag-enumeration-absent]
encounters: 1
lastSeen: 2026-07-01T22:32:08.9815698+03:00
---

# No actor-`Tags` census — you must already know a tag before you can query it

`AActor::Tags` is UE's per-instance tag array (the thing `actor.add_tag` /
`actor.remove_tag` / `actor.find_by_tag` / `actor.delete_by_tag` and the
read-only `system.inspect.find_by_tag` all operate on). Every one of those verbs
takes a **tag you already know** and matches it. There is **no verb anywhere in
the API that enumerates the distinct actor Tags that exist in a world** — no
"which tags are in use on this level?" reader.

That makes an ordinary orientation/audit intent — *"flag whether anything is
tagged for review or cleanup"* — unanswerable except by guessing. The natural
call, `system.inspect.find_by_tag {tag:"review"}`, requires the caller to name
the tag up front; a miss returns a clean empty result
(`{"objects":[],"count":0,...,"success":true}`) that is indistinguishable from
"the tag genuinely isn't present." With no way to list what tags *do* exist, the
caller can only:

- **Guess likely tag names** (`review`, `cleanup`, `Test`, `TODO`, `WIP`, …) and
  fire one `find_by_tag` per guess — which silently misses any actor tagged with
  an unguessed string (a **false-negative audit conclusion**: "nothing tagged for
  cleanup" is only as true as the guess list), or
- **Scan every actor** — `system.inspect.list_objects` for the full set, then one
  `actor.get_metadata` / `inspect_object` per actor to read its `tags` array and
  union them client-side. On the demo `ExampleProjectWelcome` level that is 227
  per-actor reads (each of which itself spills — see
  `E-inspect-object-no-projection-spills`), i.e. impractical.

So the tool is not *wrong* — `find_by_tag` correctly reports what it's asked — but
the capability to answer the phrased question ("is anything tagged for X?")
without pre-known tags is simply absent.

## What it should do

Add a read-only actor-tag census to the `system.inspect` namespace — the same
namespace/precedent as `F-inspect-list-subsystems` (which added
`system.inspect.list_subsystems` for a similar "no accessor exists for this value
space" gap). Proposed:

| Method | Params | Returns |
|---|---|---|
| `system.inspect.list_actor_tags` | `world` (`editor`/`pie`/`auto`, default `auto`, mirroring `find_by_tag`); optional `limit` + `truncated`/`totalCount` per the standard detectable-elision contract | `{ tags: [ { tag, count } ], distinctCount, world, worldPath, success }` |

One `TActorIterator<AActor>` pass unioning each actor's `Tags` (with an
occurrence `count` per tag) directly answers "what tags exist and how many actors
carry each," turning the review/cleanup audit into a single inline call instead of
a guessing game or an N-call scan. (Cheap alternative if a new verb is unwanted:
add `tags` to the `system.inspect.list_objects` per-row field set so a single
`list_objects` gives every actor + its tags and the caller unions client-side —
but a dedicated census is the direct fit.)

## Repro (live, ExampleProjectWelcome / PersistentLevel, this HEAD)

Task: *"flag whether anything is tagged for review or cleanup."* Replayed:

`system.inspect.find_by_tag {"tag":"review"}` →
```json
{"objects":[],"count":0,"world":"auto","worldPath":"/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome","success":true}
```

Empty — but with no tag-enumeration verb there is no way to tell whether "review"
is genuinely unused or simply the wrong guess, nor to discover what tags the 227
actors actually carry. The attempt agent had to fire five guesses
(`review`/`cleanup`/`Test`/`TODO`/`WIP`, all `count:0`) plus an `inspect_object`
of one injected actor to confirm its `Tags` array was really empty rather than
`find_by_tag` being broken. Verbatim friction: *"find_by_tag needs an exact tag
name with no tag-enumeration method, so verifying 'nothing tagged for
review/cleanup' required guessing likely tags (review/cleanup/Test/TODO/WIP) plus
inspecting an actor's Tags to confirm the empties were genuine rather than a
broken search."*

Guilty source (the absence): `system.inspect.find_by_tag` registers a **required**
`tag` and exact-matches it —
`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp:1550`
`RPC_PARAM_REQ("tag", "string", "Actor tag (FName) to look up in AActor::Tags.")`
and `:1573` `if (Actor->ActorHasTag(FName(*Tag)))`. No handler registers a
tag-listing verb (grep of `Handlers/` for a `list_actor_tags` / tag-enumeration
registration finds none). The read-only `system.inspect` twin doesn't even offer
the `matchType='contains'` substring broadening the `actor.find_by_tag` twin has —
it's exact-only.

## Distinct from

- `E-inspect-find-by-tag-internal-name-not-label` (IN-REVIEW) — same method, a
  different angle: that ticket is about the `name` field being the internal object
  name vs the display label on the rows `find_by_tag` **returns**; this ticket is
  that you can't discover **which tags to query in the first place**. Orthogonal.
- `gameplay_tags.list` / `F-gameplay-tags-namespace` — that lists the project's
  **GameplayTag registry** (`FGameplayTag` hierarchical tags from the tag manager),
  a completely different data model from `AActor::Tags` (an `FName` array on each
  instance). It cannot answer "what actor Tags are on this level."
- `E-inspect-list-objects-no-limit-spills` / `E-inspect-object-no-projection-spills`
  — the same task's response-spill friction on `list_objects`/`inspect_object`;
  those are already-filed size/projection ergonomics, not the missing-enumeration
  capability this ticket owns. (Note: `list_objects` rows don't carry `tags` even
  after those fixes, so they don't cover this gap either.)

severity rationale: impact=blocker-with-workaround (only unreliable tag-guessing —
which risks a silent false-negative audit conclusion — or an impractical O(N)
per-actor scan) × reach=orientation/audit path (a normal but not every-session
authoring intent) -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Filed from a realism-mode orientation task ("give me a rundown of the active world … flag whether anything is tagged for review or cleanup") on `ExampleProjectWelcome`/`PersistentLevel` (227 actors), outcome **done** with minor friction — all success checks passed, no call errored. Replay-confirmed at HEAD: `system.inspect.find_by_tag {tag:"review"}` returns a clean empty set, and there is no verb anywhere that enumerates the distinct actor `Tags` in a world, so "is anything tagged for review/cleanup?" is answerable only by guessing tag names (silently missing unguessed tags) or by an N-per-actor `get_metadata`/`inspect_object` scan (227 calls here, each spilling). Guilty absence grounded in source: `EnvironmentHandler.cpp:1550` (`RPC_PARAM_REQ("tag", ...)`) + `:1573` (`ActorHasTag(FName(*Tag))`) — required exact tag, no enumeration; the read-only twin lacks even the `matchType='contains'` the `actor.find_by_tag` twin has. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — no ticket owns actor-tag enumeration. `E-inspect-find-by-tag-internal-name-not-label` is the name-vs-label angle on the rows returned; `gameplay_tags.list`/`F-gameplay-tags-namespace` list the unrelated GameplayTag registry, not `AActor::Tags`; `E-inspect-list-objects-no-limit-spills`/`E-inspect-object-no-projection-spills` are the same task's response-spill ergonomics on other methods. Proposed: a read-only `system.inspect.list_actor_tags` census ({tag,count} per distinct tag, world-scoped, with the standard limit/truncated contract), mirroring the `F-inspect-list-subsystems` precedent; or minimally add `tags` to the `system.inspect.list_objects` per-row field set.
- `#2-implement` `IN-REVIEW` developer — Added the read-only census verb `system.inspect.list_actor_tags` (the proposed root-cause fix, mirroring the DONE `F-inspect-list-subsystems` precedent). One `TActorIterator<AActor>` pass unions each actor's `AActor::Tags` into distinct `{tag, count}` rows (count = actors carrying the tag; deduped within a single actor; `NAME_None` slots skipped), sorted count-desc then tag-asc. Response carries `tags[]`, `count` (rows returned), `distinctCount` (full untruncated distinct-tag total), `truncated`, `world`, `worldPath`, `success` — the same detectable-elision `limit`/`truncated` contract as `system.inspect.list_objects`. So the "is anything tagged for review/cleanup?" audit is now one inline call instead of tag-guessing or a 227-call per-actor scan. Handler in `Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp` (registered right after `system.inspect.find_by_tag`). Regression test `PinWright.system.inspect.list_actor_tags.EnumeratesDistinctTagsWithCounts` in `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp`: spawns three in-code transient actors, tags them so a GUID-unique tag has count 1 and a shared GUID-unique tag has count 3, asserts both census counts and the `distinctCount`/`limit=1`/`truncated` elision contract via the production handler. Fixture is fully in-code (no example-content/Lyra dependency); a missing editor world is a failure, not a skip. Not yet compiled/tested — leaving for the verify phase.
