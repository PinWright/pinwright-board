---
id: F-scene-composition-histogram
title: "No scene-composition / class-histogram primitive — 'what is this level made of' forces a full list_objects row dump + external aggregation"
status: IN-REVIEW
severity: Medium
category: feature
tags: [scene-composition-histogram, inspection, get_scene_stats, list_objects, aggregation, orientation]
encounters: 1
lastSeen: 2026-07-01T22:37:09.9601693+03:00
---

# No per-class actor breakdown — the "survey what's in this level" intent has no aggregate reader

Answering the extremely common orientation question — *"give me a survey of what's
actually in the scene / its composition"* — has **no aggregate primitive** in the
API. The two candidate readers each fall short:

- `system.inspect.get_scene_stats` returns **only** a grand total:
  `{"actorCount":227,"success":true}`. Its own registration doc concedes the gap —
  *"Currently reports actorCount only — extended metrics are reserved for future
  expansion."* There is no per-class map.
- `system.inspect.list_objects` returns **every actor row** (name/path/class) with
  no aggregation. On the demo `ExampleProjectWelcome`/`PersistentLevel` (227 actors)
  the array is **108306 chars**, crossing the 10000-char display threshold and
  spilling to a `Saved/.../HttpResponses/<uuid>.json` file.

So to answer "what is the scene made of," the caller must enumerate all 227 rows,
Read the spilled file, and **hand-roll a class histogram in an external shell** —
in this task, PowerShell `[regex]::Matches($raw, ...\"class\": ...) | Group-Object
| Sort Count`, yielding `112 DynamicMeshActor / 12 Landscape / 7 PointLight / 2
StaticMeshActor / …`. Even the documented narrowing levers being added to
`list_objects` (`filter`/`limit`/`namesOnly`/`fields`, see
`E-inspect-list-objects-no-limit-spills`) do **not** close this gap: they trim
rows, but 227 rows still spill and still require the caller to aggregate
client-side — **none of them produce counts-by-class**.

## What it should do

Add a dedicated read-only scene-composition census to the `system.inspect`
namespace — the class counterpart to the just-landed `system.inspect.list_actor_tags`
(the tag census) and the `system.inspect.list_subsystems` precedent, resolving the
identical *"no aggregate reader for a value space"* shape. Proposed:

| Method | Params | Returns |
|---|---|---|
| `system.inspect.list_actor_classes` | `world` (`editor`/`pie`/`auto`, default `auto`, mirroring `find_by_class`); optional `limit` + `distinctCount`/`truncated` per the standard detectable-elision contract | `{ classes: [ { class, count } ], count, distinctCount, truncated, world, worldPath, success }` |

One `TActorIterator<AActor>` pass tallying `GetClass()->GetName()` into a `TMap`
directly answers "what is this level made of," turning the ubiquitous survey into a
**single inline call** instead of a 108KB `list_objects` spill + a Read + an external
regex/Group-Object. `class` is the leaf class name that `list_objects` / `find_by_class`
already report, so a census row feeds straight back into `find_by_class` to enumerate
the actors behind a count. The verb shares one shape/limit/elision contract with
`system.inspect.list_actor_tags` (rows sorted count-desc then name-asc; `distinctCount`
reports the full untruncated distinct-class total; `truncated` flips when `limit`
elides rows) so the two censuses stay consistent.

**Why a dedicated verb, not the two shapes originally proposed here:** folding an
unbounded `classCounts` map into `get_scene_stats` would give that "basic grand-total"
reader a second, contract-less return shape (no `limit`/`truncated`); and adding a
`groupByClass` mode to `list_objects` would overload one verb with two divergent return
shapes — which the `E-inspect-list-objects-no-limit-spills` row-narrowing work
deliberately avoided. A dedicated census verb matches the established
`list_actor_tags` / `list_subsystems` precedent instead.

## Evidence

From a realism-mode orientation task (focus `null`, namespace `system`, outcome
**done** — every call first-try; judge filed the tag-enumeration gap as
`F-inspect-list-actor-tags`). Story step: *"a survey of what's actually in the
scene — list the actors so I can see its makeup."* CallAnalyzer call-trace
finding, verbatim:

> "There is no per-class breakdown primitive: get_scene_stats returns only
> actorCount (227) — its wiki says 'Currently reports actorCount only — extended
> metrics are reserved for future expansion.' To answer composition the agent
> called system.inspect.list_objects with args={}, which returned
> {"outputTooLong":true,...108306 chars...spilled to file}, then had to hand-roll
> a class histogram in PowerShell: [regex]::Matches($raw,'...class...') |
> Group-Object. Even with the documented narrowing levers (namesOnly/fields), 227
> rows still spill AND still require external aggregation — none of the levers
> produce counts-by-class."

## Distinct from

- `E-inspect-list-objects-no-limit-spills` (IN-REVIEW) — adds `filter`/`limit`/
  `namesOnly`/`fields` to trim the **row** output; it never aggregates, so
  "counts by class" is still a client-side job after that fix. Orthogonal:
  narrowing rows vs. summarizing them.
- `B-inspect-settings-stats-stub-silent-success` (IN-REVIEW) — about the *other*
  four `system.inspect` stat readers (`get_editor_settings` etc.) being dishonest
  no-op stubs; it explicitly **excludes** `get_scene_stats` (which returns real
  `actorCount`). This ticket leaves `get_scene_stats` untouched and adds a *separate*
  composition census verb — a capability add, orthogonal to that stub-honesty fix.
- `F-inspect-list-actor-tags` (IN-REVIEW — filed the same task) — the missing
  **tag**-census; this is the missing **class**-census. Same "no aggregate reader
  for a value space" shape, different value space. Its dedicated
  `system.inspect.list_actor_tags` verb (`{tags:[{tag,count}], distinctCount,
  truncated, world, worldPath, success}`) is the precedent this ticket now mirrors as
  `system.inspect.list_actor_classes`.

severity rationale: impact=soft-blocker (doable only via a 108KB `list_objects`
spill + a file Read + external regex/Group-Object aggregation) × reach=every
orientation/survey opens with "what's in this level" (common, not literally every
session) -> Medium

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a realism-mode orientation task (focus `null`, namespace `system`, outcome **done**; judge filed the sibling tag-census gap as `F-inspect-list-actor-tags`). PROCESS friction: the "survey what's in this scene" intent has no aggregate primitive — `system.inspect.get_scene_stats` returns only `{actorCount:227}` (docs: "reports actorCount only — extended metrics are reserved for future expansion") and `system.inspect.list_objects {}` returned 108306 chars → spilled to a HttpResponses file, forcing the agent to hand-roll a class histogram in PowerShell (`[regex]::Matches(...class...) | Group-Object`) → `112 DynamicMeshActor / 12 Landscape / 7 PointLight / 2 StaticMeshActor / …`. The `list_objects` narrowing levers being added by `E-inspect-list-objects-no-limit-spills` (IN-REVIEW) trim rows but never aggregate, so counts-by-class stays a client-side job. Proposes extending `get_scene_stats` with a `classCounts` histogram (its docs already reserve "extended metrics") and/or a `groupByClass` mode on `list_objects`. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — no ticket owns a class-histogram/scene-composition reader (`B-inspect-settings-stats-stub-silent-success` excludes `get_scene_stats`; `E-inspect-list-objects-no-limit-spills` narrows rows without aggregating; `E-inspect-object-class-key-drift` is the class-key naming angle on inspect_object).
- `#2-reword-and-implement` `IN-REVIEW` developer — Reworded the fix shape (validity review returned 2× reword) from the ticket's two original options (fold a `classCounts` map into `get_scene_stats` / add a `groupByClass` mode to `list_objects`) to a dedicated read-only census verb, matching the just-landed `system.inspect.list_actor_tags` (IN-REVIEW) and `system.inspect.list_subsystems` (DONE) "no aggregate reader for a value space" precedent — an unbounded `classCounts` on the grand-total reader would be contract-less (no `limit`/`truncated`) and a `list_objects` `groupByClass` mode would give one verb two divergent shapes. Implemented `system.inspect.list_actor_classes`: one `TActorIterator<AActor>` pass tallies each actor's `GetClass()->GetName()` (the same leaf name `list_objects`/`find_by_class` report, so a row feeds back into `find_by_class`) into a `TMap<FString,int32>`, emitting distinct `{class, count}` rows sorted count-desc then class-asc. Response carries `classes[]`, `count` (rows returned), `distinctCount` (full untruncated distinct-class total), `truncated`, `world`, `worldPath`, `success` — the same `limit`/`truncated` detectable-elision contract as `list_actor_tags`. So "what is this level made of?" is one inline call instead of a 108KB `list_objects` spill + Read + external Group-Object. Handler in `Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp` (registered right after `system.inspect.list_actor_tags`). Regression test `PinWright.system.inspect.list_actor_classes.CountsActorsByClass` in `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp`: captures a baseline census, spawns 3 transient `AStaticMeshActor`s + 1 bare `AActor` into the editor world, and asserts the census counts rose by exactly +3 for `StaticMeshActor` and +1 for `Actor` (a delta, since the open level may already carry those classes — different deltas prove it groups BY class, not a grand total), plus the `distinctCount`/`limit=1`/`truncated` elision contract, all via the production handler. Fixture is fully in-code (no example-content/Lyra dependency); a missing editor world is a failure, not a skip. Not yet compiled/tested — leaving for the verify phase.
