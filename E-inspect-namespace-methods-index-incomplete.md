---
id: E-inspect-namespace-methods-index-incomplete
title: "system.inspect.* readers register Category 'system', so their `## Methods` rows index on the `system` page, not the `system.inspect` page their names imply (16 methods split off); fix = align their Category to system.inspect"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [namespace-index-incomplete, wiki, discoverability, system-inspect, category-split]
encounters: 1
lastSeen: 2026-07-11T22:18:30.9536453+03:00
claimedBy: fuzz2
claimedAt: 2026-07-12T21:11:19.4377943+03:00
---

# The `system.inspect.*` scene/inspection readers register Category `system`, so they land on the `system` page's `## Methods` index — not on the `system.inspect` page their names imply

## Mechanism (verified in source)

The auto-generated `## Methods` index on a namespace page is built by **Category**, not by
method-name prefix: `RenderMethodList` looks up `Cache.MethodsByCategory.Find(PathLower)`
(`Catalog/WikiHandler.cpp:357`), and that map is keyed by `Reg.Category.ToLower()`
(`WikiHandler.cpp:140-146`). Each method page's `Namespace:` line is likewise
`Reg.Category.ToLower()` (`WikiHandler.cpp:371`).

Sixteen methods whose NAMES are `system.inspect.*` register **Category `"system"`** in
`Handlers/Environment/EnvironmentHandler.cpp` (the legacy handler file), while the eight
game-state/subsystem readers correctly register **Category `"system.inspect"`**
(`SystemInspectSingletonsHandler.cpp`, `SubsystemInspectHandler.cpp:31`,
`ClassSearchHandler.cpp:110`). So the served `system.inspect` page's `## Methods` index
(`Saved/PinWright/wiki/system.inspect.md:20-27`) lists only those 8, and the 16
`system.inspect.*` methods render on the **`system`** page's index instead
(`Saved/PinWright/wiki/system.md:26-41`).

The 16 (all `EnvironmentHandler.cpp`, currently Category `"system"`):

- **12 genuine readers**: `get_world_settings` (:1271), `get_viewport_info` (:1291),
  `get_selected_actors` (:1322), `get_scene_stats` (:1351), `list_objects` (:1391),
  `find_by_class` (:1533), `find_objects_by_class` (:1582), `find_by_tag` (:1704),
  `list_actor_tags` (:1809), `list_actor_classes` (:1866), `inspect_class` (:1907),
  `inspect_object` (:1998).
- **4 NOT_IMPLEMENTED stubs** (they fail honestly with `NOT_IMPLEMENTED` per
  `B-inspect-settings-stats-stub-silent-success`): `get_project_settings` (:1249),
  `get_editor_settings` (:1260), `get_performance_stats` (:1371), `get_memory_stats` (:1381).

## Impact (corrected)

This is a name-vs-Category split, **not** an undiscoverability hole. The 16 ARE indexed and
reachable — on the `system` page (`call("system")`), each with its full dotted name and
summary; e.g. `list_actor_tags` is at `system.md:40` with an accurate "distinct AActor::Tags
in use" summary. The defect is the internal inconsistency: `system.inspect` advertises itself
as the read-only inspection namespace, yet its `## Methods` index omits the actual inspection
verbs, and each of the 16 method pages shows the wrong `Namespace: system`. A caller reasoning
from the method name (`system.inspect.list_actor_tags` -> look on the `system.inspect` page)
gets a violated expectation — the friction the reporter's level-census audit hit (it missed
`list_actor_tags` while skimming `system.inspect.md`, not having consulted the parent
`system.md` index where the verb is listed). Reach is every orientation/census session that
skims `system.inspect`, so this stays Medium despite the corrected (smaller) impact.

## Fix (source — align Category with the method-name prefix)

Change arg2 of the 16 `REGISTER_RPC_HANDLER("system.inspect.<verb>", "system", ...)` calls in
`Handlers/Environment/EnvironmentHandler.cpp` from `"system"` to `"system.inspect"`. This is a
SOURCE change, **not** a `Docs/wiki-src/` overlay edit — no overlay can add rows to the
registry/Category-sourced auto `## Methods` index (`WikiHandler.cpp:357`). Consequences, all
improvements: the 16 move onto the `system.inspect` index (their name's page), each method
page's `Namespace:` line corrects to `system.inspect`, and the `system` index sheds its 16
stray `system.inspect.*` rows (keeping its 9 genuine verbs — `call_subsystem`,
`console_command`, `job_*`, `live_coding_*`, `run_tests`, `run_ubt` — so `system` stays a
valid Hybrid node). Dispatch is by full method name and is untouched; the `system.inspect`
tree node already exists, so no tree change.

Flip **all 16** (including the 4 NOT_IMPLEMENTED stubs): their names are `system.inspect.*`, so
leaving any behind would recreate the same split for those verbs (still Category `system`,
still `Namespace: system` on their page). The stubs' honest NOT_IMPLEMENTED summaries render
fine on the index, and `B-inspect-settings-stats-stub-silent-success` keeps the registrations,
so this does not conflict with that ticket.

## Dropped from the original report

- The secondary "add gameplay-tags vocabulary to the `list_actor_tags` page" ask is factually
  wrong and is NOT implemented: `list_actor_tags` reads `AActor::Tags` (`TArray<FName>`), a
  DIFFERENT UE system from `FGameplayTag` / `FGameplayTagContainer` (served by
  `gameplay_tags.list`). Adding "gameplay tags" wording would conflate the two and steer a
  gameplay-tag question to an AActor::Tags census. The existing summary ("distinct AActor::Tags
  in use ... discover which tags exist") is accurate; the Category fix that puts
  `list_actor_tags` on the `system.inspect` index is the real discoverability improvement.

## Distinct from

- `F-inspect-list-actor-tags` (IN-REVIEW) — the FEATURE that *added*
  `system.inspect.list_actor_tags`. The method now exists and works; this ticket is the
  orthogonal categorization gap. Not blocked by it.
- `E-environment-control-subnamespace-no-index-page` (IN-REVIEW) — same Category-vs-method-name
  mechanism, different symptom: that one advertises a `call()` handle (`environment.control`)
  that resolves to a did-you-mean list; this one is a namespace page whose `## Methods` index
  omits registered, dispatching sibling methods. Unlike that case, `system.inspect` is already
  a real, populated Category node, so the fix here is a mechanical arg2 edit with no
  WikiHandler/generator/tree churn.
- `F-scene-composition-histogram` (landed) — added `list_actor_classes`, itself one of the 16
  omitted from the `system.inspect` index and fixed here.
- `E-inspect-find-by-class-no-limit-spills` (OPEN) — the same task's `find_by_class` response
  spill. Orthogonal size ergonomics, not categorization.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from a clean read-only level-census struggle audit (namespace `system`, 10 calls, all ok, outcome `done`). Ground truth: the served `system.inspect` namespace page's `## Methods` index (`Saved/PinWright/wiki/system.inspect.md` lines 18-27) lists only 8 game-state/subsystem readers (get_game_instance/mode/state, get_local_players, get_player_controllers, get_player_states, list_subsystems, search_classes) and omits every actor/scene-inspection reader — get_scene_stats, list_actor_classes, find_by_class, find_by_tag, list_actor_tags, list_objects, inspect_object, inspect_class, get_world_settings, get_selected_actors, get_viewport_info — although each has its own generated method page and dispatches fine. Likely cause: those readers register Namespace/Category `system` (confirmed: `system.inspect.list_actor_tags.md` line 5 = `Namespace: system`) while the auto `## Methods` index emits by Category `system.inspect`, so the scene readers scatter off the page (same Category-vs-method-name mechanism as `E-environment-control-subnamespace-no-index-page`). Consequence: the agent ran ~8 tag-oriented wiki reads/greps (case-sensitive index grep for `gameplay tag|GameplayTag|tags in use|tag census`, a `tag` grep of system.inspect.md, reads of gameplay_tags/actor.find_by_tag/actor.list/system.inspect pages) and STILL missed `system.inspect.list_actor_tags`, then reported a false "no scene-wide gameplay-tags-in-use helper exists" capability-gap and answered the tag census from one representative actor's empty `tags` field — an approach that under-reports on any tagged level; the task passed only because this level has zero actor tags and the success check allows an empty census. Secondary miss: the list_actor_tags page says "AActor::Tags in use" (no `gameplay tag` string, capitalized "Tags") so the task-vocabulary/case grep never matched it. Distinct from `F-inspect-list-actor-tags` (the feature that added the verb; not blocked by it — method + page already on disk), `E-environment-control-subnamespace-no-index-page` (advertised-handle did-you-mean, same mechanism/different page), `F-scene-composition-histogram` (added list_actor_classes, itself omitted from the index), and `E-inspect-find-by-class-no-limit-spills` (the same task's spill). Proposed docs fix: surface the scene readers in `Docs/wiki-src/system.inspect.md`'s index (register them under Category `system.inspect` so the auto-index picks them up, or add an explicit cross-link list + name list_actor_tags as the find_by_tag census counterpart), and add the caller's tag vocabulary ("gameplay tags in use across actors", "per-actor tag census") to `Docs/wiki-src/system.inspect.list_actor_tags.md`.
- `#2-reword` `IN-REVIEW` developer — REWORD + GO. Verified in source that the defect is real (the `## Methods` index is Category-keyed: `WikiHandler.cpp:357` -> `MethodsByCategory[Reg.Category.ToLower()]` at :140-146), and that 16 methods NAMED `system.inspect.*` register Category `"system"` in `EnvironmentHandler.cpp` (12 genuine readers + 4 NOT_IMPLEMENTED stubs) while the 8 game-state/subsystem readers register `"system.inspect"` — so the 16 index on `system.md:26-41`, not `system.inspect.md:20-27`. Reworded the ticket to reality: it had mislabeled a source-level Category change as a `Docs/wiki-src/` overlay edit (no overlay can touch the registry-sourced index), under-scoped it (16, not ~11, incl. the 4 stubs), and overstated impact as "undiscoverable / false capability-gap" (the 16 ARE indexed, on `system.md`). Intended scope: flip arg2 `"system"` -> `"system.inspect"` for all 16 registrations in `EnvironmentHandler.cpp`; keep severity Medium / category ergonomic; `system` stays a Hybrid node with its 9 genuine verbs. Dropped the secondary "add gameplay-tags vocabulary to the list_actor_tags page" ask as factually wrong and will-not-ship (list_actor_tags reads `AActor::Tags` / `TArray<FName>`, not `FGameplayTag`). Adopting the red test `PinWright.infra.wiki_handler.Namespace.SystemInspectSceneReadersIndex`.
