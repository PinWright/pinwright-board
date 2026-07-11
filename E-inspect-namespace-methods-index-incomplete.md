---
id: E-inspect-namespace-methods-index-incomplete
title: "system.inspect namespace page's ## Methods index omits every scene-inspection reader (get_scene_stats, find_by_class, list_actor_classes, list_actor_tags, ...) — they register Namespace 'system', not 'system.inspect', so a census/orientation agent can't discover them and emits a false capability-gap"
status: OPEN
severity: Medium
category: ergonomic
tags: [namespace-index-incomplete, docs, wiki, discoverability, system-inspect, list-actor-tags]
encounters: 1
lastSeen: 2026-07-11T22:18:30.9536453+03:00
---

# The `system.inspect` namespace page's `## Methods` index lists only the 8 game-state/subsystem readers and omits every actor/scene-inspection reader — so the core orientation verbs are undiscoverable from their own namespace page

The served `system.inspect` namespace page (`Saved/PinWright/wiki/system.inspect.md`,
regenerated from `Docs/wiki-src/system.inspect.md`) opens with rich prose about
`list_objects` / `find_by_class` / `find_by_tag` / `inspect_object` / `get_scene_stats`,
but its auto-emitted `## Methods` index (the skimmable list an agent greps to find the
right verb) contains **only 8 rows, all game-state / subsystem**:

```
get_game_instance, get_game_mode, get_game_state, get_local_players,
get_player_controllers, get_player_states, list_subsystems, search_classes
```

It **omits every actor/scene-inspection reader** — including the six the audited agent
actually used (`get_scene_stats`, `list_actor_classes`, `find_by_class`,
`get_world_settings`, `get_selected_actors`, `get_viewport_info`) plus `list_objects`,
`inspect_object`, `inspect_class`, `find_by_tag`, and the newly-landed
`list_actor_tags`. Every one of those has its own generated method page
(`system.inspect.find_by_class.md`, `system.inspect.get_scene_stats.md`,
`system.inspect.list_actor_tags.md`, ...), so they are registered and dispatch fine —
they are simply **absent from the index on the very page a caller lands on to find them**.

## Likely root cause — Namespace/Category ≠ method-name prefix

The `## Methods` index is auto-emitted from the handler registry by **Category**, not by
method-name prefix (same Category-driven tree mechanism documented in
`E-environment-control-subnamespace-no-index-page`). The scene-inspection readers have
method NAMES `system.inspect.*` but register a **different Namespace/Category**:
`system.inspect.list_actor_tags.md` line 5 literally reads `Namespace: system` (not
`system.inspect`). So the auto-index on the `system.inspect` page lists only methods whose
Category is exactly `system.inspect` (the 8 game-state readers), and the scene readers —
Category `system` — scatter off the page. This is a registration/Category split surfacing
as a docs discoverability hole; it is not a router failure (the methods all dispatch by
full name).

## Second discoverability miss — `list_actor_tags` wording evades the task vocabulary

Even reaching the `list_actor_tags` method page directly is hard from the task's words.
The goal asked for "which **gameplay tags** are in use across those actors." The page
(`system.inspect.list_actor_tags.md`) describes "distinct **AActor::Tags** in use" and
"the census find_by_tag can't give you" — accurate, but it contains **no `gameplay tag`
string**, and its capitalized "Tags in use" evades a case-sensitive grep for
"tags in use". So a vocabulary/case search on the task's phrasing never matches it.

## Impact (this task)

Struggle-audit of a clean, read-only level-census task (namespace `system`, 10 calls,
all ok, outcome `done`, judge filed only the separate `find_by_class` spill). The agent
ran ~8 tag-oriented wiki reads/greps — a case-sensitive index-wide grep for
`gameplay tag|GameplayTag|tags in use|tag census` (19 files, `list_actor_tags` not among
the hits), a grep of `system.inspect.md` for `tag`, and reads of `gameplay_tags.md`,
`gameplay_tags.list.md`, `actor.find_by_tag.md`, `actor.list.md`, `system.inspect.md` —
and **still missed** `system.inspect.list_actor_tags`, the exact-fit census verb. It then
reported a FALSE capability-gap in its friction note:

> there is no single scene-wide "gameplay tags in use across actors" helper — find_by_tag
> is query-by-tag and gameplay_tags.list is the project registry, so per-actor tags only
> come from actor.describe

and answered the tag census only by reading one representative actor's (empty) `tags`
field — an approach that would **under-report on any level that actually has actor tags**.
The task passed only because this level has zero actor tags and the success check allows
an empty tag census. On a tagged level the same discoverability hole would have produced a
wrong (incomplete) census that the caller trusts.

## Fix (docs / overlay — downstream)

Surfaces to improve (both under `Docs/wiki-src/`):

- **`docs/wiki-src/system.inspect.md`** — make the namespace page surface its scene-inspection
  readers in the skimmable index. Either (a) register the scene readers under Category
  `system.inspect` so the auto `## Methods` index picks them up (aligns Namespace with the
  method-name prefix), or (b) if the Category split is deliberate, add an explicit prose
  list / cross-link section on the `system.inspect` overlay naming `get_scene_stats`,
  `list_actor_classes`, `find_by_class`, `find_by_tag`, `list_actor_tags`, `list_objects`,
  `get_world_settings`, `get_selected_actors`, `get_viewport_info`, `inspect_object`,
  `inspect_class` — and name `list_actor_tags` as the scene-wide actor-tag census
  counterpart to `find_by_tag` in the "Cross-cluster overlap" block.
- **`docs/wiki-src/system.inspect.list_actor_tags.md`** (or a `### system.inspect.list_actor_tags`
  H3 in the namespace overlay) — add the search vocabulary a caller actually types:
  "gameplay tags in use across actors", "per-actor tag census", lowercase "which tags are
  in use", so a vocabulary/case grep on the task phrasing lands on it. Its current
  "AActor::Tags in use" wording matches none of those.

## Distinct from

- `F-inspect-list-actor-tags` (IN-REVIEW) — the FEATURE that *added* `system.inspect.list_actor_tags`.
  The method now exists and works; this ticket is the orthogonal **discoverability** gap
  that the same census verb is unfindable from the namespace index and the tag vocabulary.
  Not blocked by it (the method + its page already exist on disk; the namespace-index
  omission is independent of the feature).
- `E-environment-control-subnamespace-no-index-page` (IN-REVIEW) — same Category-vs-method-name
  mechanism, different page: that one advertises a `call()` handle (`environment.control`)
  that resolves to a did-you-mean list; this one is a namespace page whose `## Methods`
  index silently omits registered, working sibling methods.
- `F-scene-composition-histogram` (landed) — added `list_actor_classes`, which the agent
  used successfully. This ticket is that `list_actor_classes` (and its scene-reader
  siblings) are missing from the `system.inspect` `## Methods` index.
- `E-inspect-find-by-class-no-limit-spills` (OPEN) — the same task's `find_by_class` response
  spill (judge already dedup-bumped). Orthogonal size ergonomics, not discoverability.

severity rationale: impact=docs/discoverability (Low) × reach=every-session — the
`system.inspect` namespace page is a primary orientation/census entry point and its
`## Methods` index omits ~11 of the core read-only scene verbs, so most census/orientation
tasks that skim it are affected -> bump up one -> Medium. Blocked task: the level-census /
"which gameplay tags are in use" orientation task; workaround cost: ~8 wasted wiki
reads/greps, a false capability-gap conclusion, and answering the tag census from one
representative actor's tags (which silently under-reports on any tagged level).

## History
- `#1-initial-audit` `OPEN` reporter — Filed from a clean read-only level-census struggle audit (namespace `system`, 10 calls, all ok, outcome `done`). Ground truth: the served `system.inspect` namespace page's `## Methods` index (`Saved/PinWright/wiki/system.inspect.md` lines 18-27) lists only 8 game-state/subsystem readers (get_game_instance/mode/state, get_local_players, get_player_controllers, get_player_states, list_subsystems, search_classes) and omits every actor/scene-inspection reader — get_scene_stats, list_actor_classes, find_by_class, find_by_tag, list_actor_tags, list_objects, inspect_object, inspect_class, get_world_settings, get_selected_actors, get_viewport_info — although each has its own generated method page and dispatches fine. Likely cause: those readers register Namespace/Category `system` (confirmed: `system.inspect.list_actor_tags.md` line 5 = `Namespace: system`) while the auto `## Methods` index emits by Category `system.inspect`, so the scene readers scatter off the page (same Category-vs-method-name mechanism as `E-environment-control-subnamespace-no-index-page`). Consequence: the agent ran ~8 tag-oriented wiki reads/greps (case-sensitive index grep for `gameplay tag|GameplayTag|tags in use|tag census`, a `tag` grep of system.inspect.md, reads of gameplay_tags/actor.find_by_tag/actor.list/system.inspect pages) and STILL missed `system.inspect.list_actor_tags`, then reported a false "no scene-wide gameplay-tags-in-use helper exists" capability-gap and answered the tag census from one representative actor's empty `tags` field — an approach that under-reports on any tagged level; the task passed only because this level has zero actor tags and the success check allows an empty census. Secondary miss: the list_actor_tags page says "AActor::Tags in use" (no `gameplay tag` string, capitalized "Tags") so the task-vocabulary/case grep never matched it. Distinct from `F-inspect-list-actor-tags` (the feature that added the verb; not blocked by it — method + page already on disk), `E-environment-control-subnamespace-no-index-page` (advertised-handle did-you-mean, same mechanism/different page), `F-scene-composition-histogram` (added list_actor_classes, itself omitted from the index), and `E-inspect-find-by-class-no-limit-spills` (the same task's spill). Proposed docs fix: surface the scene readers in `Docs/wiki-src/system.inspect.md`'s index (register them under Category `system.inspect` so the auto-index picks them up, or add an explicit cross-link list + name list_actor_tags as the find_by_tag census counterpart), and add the caller's tag vocabulary ("gameplay tags in use across actors", "per-actor tag census") to `Docs/wiki-src/system.inspect.list_actor_tags.md`.
