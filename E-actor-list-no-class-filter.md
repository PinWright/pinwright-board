---
id: E-actor-list-no-class-filter
title: "actor.list has no class filter and doesn't steer class-intent callers to actor.find_by_class — they guess classFilter then fall back to a name-substring proxy"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [actor, docs, discovery, filter, class-name]
---

# actor.list has no class filter; class-intent callers guess `classFilter` then abuse `filter`

`actor.list` is the obvious "enumerate the actors" entry point, but its only
narrowing param is `filter` — a **substring match against the actor label/name**
(`actor.list.md`: *"Substring filter applied to actor label and name
(case-sensitive)"*). A caller whose intent is class-shaped ("list every
PostProcessVolume in the level") has no class param to reach for, so the natural
guess is `classFilter` — which the dispatcher rejects with
`[UNKNOWN_PARAMS] ... Valid parameters: [filter, world]`. The caller then falls
back to `filter="PostProcessVolume"`, a **name-substring** match that only
*coincidentally* works because the actors carry default labels
(`PostProcessVolume_0`, `…_1`) that happen to contain the class string. Rename or
relabel the actors and the same call silently returns the wrong set — a latent
footgun, not a real class filter.

The correct tool for the class intent is the sibling `actor.find_by_class`
(`className` accepts a short class name and matches subclasses). But nothing on
the `actor.list` path points there: the `actor.list` wiki page steers only to
`system.inspect.list_objects` ("for object-graph traversal … prefer …"), and the
`actor` namespace overlay's "Cross-cluster overlap" section pairs `actor.list` /
`actor.find_by_class` / `actor.find_by_tag` with their `system.inspect.*` twins
but never disambiguates **which of the three to use for which narrowing intent**
(name substring → `find_by_name`/`list filter=`; class → `find_by_class`; tag →
`find_by_tag`). So a class-intent caller landing on the obvious `actor.list` has
no on-page cue to switch methods, and discovers the dead end only by eating the
`UNKNOWN_PARAMS` round-trip.

This is the same class/filter-naming friction `E-material-editor-param-name-drift`
flagged for `actor.list` (`filter`) but explicitly held **out of scope** there
(*"a related class/filter naming gap, not a path-vs-assetPath case — out of scope
here"*), and a cousin of `E-volume-type-filter-discovery` (same "name `filter` vs
class filter" confusion, but on `volume.get_volumes_info`, which at least *has* a
`volumeType` param). It is distinct from `B-actor-find-by-class-short-name-fails`
(IN-REVIEW) — that is the *bug* in the alternative method; this is the
*discoverability* friction that keeps callers on `actor.list` in the first place.

## What it should do

Docs-first (cheapest, no behavior change). The overlay already exists at
`docs/wiki-src/actor.md` (it carries the `## Cross-cluster overlap` section with
a `**Read-only audit**` bullet plus `### actor.describe` / `### actor.get_components`
/ etc. H3 sections). The `actor.md` namespace page renders the overlay's prelude
+ `##` sections + the auto-generated method index, and `### actor.list` surfaces
only when an agent calls `actor.list` directly — there is **no** separate
`actor.list.md` overlay. So this is an **edit to the existing overlay**, not a
new-file creation:

1. Add an `### actor.list` H3 section to the existing `docs/wiki-src/actor.md`
   (none exists today) that states `filter` is a **name/label substring** match,
   NOT a class filter — and that it only *coincidentally* works on default labels
   like `PostProcessVolume_0`, so a rename/relabel silently breaks it. To narrow
   by actor class, call `actor.find_by_class` (short class name, matches
   subclasses) instead.
2. Extend the existing "Cross-cluster overlap" → "Read-only audit" bullet (or add
   a sibling bullet under that `##` section) to disambiguate the three `actor.*`
   queries by *narrowing intent*: name substring → `actor.find_by_name` /
   `actor.list filter=`; class (incl. subclasses) → `actor.find_by_class`; tag →
   `actor.find_by_tag`. The existing bullet only maps each to its
   `system.inspect.*` read-only twin, which doesn't help a caller choosing
   *between* them.

Optional ergonomic follow-up (out of this docs scope): accept a `class` /
`classFilter` alias on `actor.list` (or have the `UNKNOWN_PARAMS` message for a
`class`-ish unknown param suggest `actor.find_by_class`), so the natural guess
self-corrects instead of dead-ending.

## Evidence

From the golden-hour post-process grade task (namespace `post_process`,
outcome `clean`). Friction note, verbatim: *"Minor: guessed actor.list param name
(classFilter) which the MCP rejected with a clear UNKNOWN_PARAMS error listing
valid params; one wiki-nav read fixed it (use filter substring)."* Call-log:
`actor.list {classFilter:"PostProcessVolume"}` → `[UNKNOWN_PARAMS] ... Valid
parameters: [filter, world]` (1 rejected call) → `actor.list.md` wiki-nav (1
read) → `actor.list {filter:"PostProcessVolume", world:"editor"}` (worked, but
only because the volumes carry default `PostProcessVolume_*` labels). The
intended target was the *class* PostProcessVolume — `actor.find_by_class` was the
right tool but was never surfaced on the `actor.list` path. Net: one wasted
round-trip + one wiki read for a "list all actors of class X" intent, and the
working fallback is a name-substring proxy that breaks under relabeling.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the post_process golden-hour struggle audit (outcome clean). `actor.list` offers no class filter, so a class-intent caller guesses `classFilter` (UNKNOWN_PARAMS, 1 wasted call), reads `actor.list.md` (1 nav), then falls back to `filter="PostProcessVolume"` — a label/name substring that only works because the actors keep default class-derived labels. The right tool, `actor.find_by_class`, is never cross-referenced from the `actor.list` wiki page or the `actor` namespace overlay's read-only-audit bullet (which only maps to the `system.inspect.*` twins). Proposed: docs-first — add an `### actor.list` section to a new `docs/wiki-src/actor.md` overlay clarifying `filter` is name-only and pointing class intents at `actor.find_by_class`, and extend the namespace cross-cluster bullet to disambiguate name/class/tag narrowing. Deliberately scoped out of `E-material-editor-param-name-drift #1` ("out of scope here"); cousin of `E-volume-type-filter-discovery`; distinct from `B-actor-find-by-class-short-name-fails` (the bug in the alternative method, not this discoverability gap).
- `#2-reword` `OPEN` lead-engineer — Corrected a material factual error in the body before implementing. The reporter's "no overlay exists yet — the `actor.md` namespace page and `actor.list.md` are generated purely from handler registrations" and "add ... to a **new** `docs/wiki-src/actor.md` overlay" are false: `docs/wiki-src/actor.md` already exists (verified against the live tree — 46-line hand-authored overlay with the `## Cross-cluster overlap` → `**Read-only audit**` bullet and `### actor.describe`/`### actor.get_components`/etc. H3 sections; no `### actor.list` section yet; no separate `actor.list.md` overlay). Reworded "What it should do" to frame the work as an EDIT to the existing overlay (add the `### actor.list` H3, extend the existing read-only-audit bullet), not a new-file creation. Defect, repro string, and the two proposed change contents are otherwise correct and confirmed in `Handlers/Actor/QueryHandler.cpp` (`actor.list` registers only `filter`+`world`, `filter` is a label/name `Contains` substring; `actor.find_by_class` resolves short names via `ResolveUClass` and matches subclasses via `TActorIterator<AActor>(World, ClassToFind)`).
- `#3-docs-fix` `IN-REVIEW` lead-engineer — Implemented the reworded docs-first fix in the plugin clone overlay `Plugins/EditorAutomationRpcGateway/docs/wiki-src/actor.md`: (1) appended an `### actor.list` H3 section stating `filter` is a name/label substring (NOT a class filter), warning it only coincidentally matches default `PostProcessVolume_*`-style labels and silently breaks under relabeling, and routing class-intent callers to `actor.find_by_class` (short class name, matches subclasses); (2) added a sibling `**Narrowing intent**` bullet under `## Cross-cluster overlap` disambiguating the three `actor.*` queries — name substring → `actor.find_by_name` / `actor.list filter=`; class (incl. subclasses) → `actor.find_by_class`; tag → `actor.find_by_tag`. No code/behavior change. Regression test: `FWikiHandlerActorListDocumentsNameNotClassFilterTest` in `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` renders `actor.list` and `actor` via `WikiHandler::RenderPage` and asserts on overlay-exclusive markers (the `### actor.list` body and the new narrowing-intent bullet) that appear in no handler registration summary, so it fails iff the overlay edit is reverted.
- `#4-evidence-namefilter-spelling` `IN-REVIEW` reporter — Cross-task aggregation,
  a different wrong guess on the SAME `actor.list` narrowing slot: the torch-lit-
  dungeon preview-lighting struggle audit (namespace `effect`; outcome `clean`; no
  judge filing). Listing the preview lights (a **name/label-substring** intent, the
  case `filter` *does* serve), the agent guessed `actor.list {nameFilter:"Light"}`
  → `[UNKNOWN_PARAMS] Unknown parameter(s) for 'actor.list': [nameFilter]. Valid
  parameters: [filter, world, limit, fields, namesOnly]. Call 'actor.list' with no
  'args' field to fetch its wiki page.`, corrected on retry with `{filter:"Light"}`
  (success). One wasted `is_error` call, zero blocked progress. Two facts distinct
  from `#1`–`#3`: (a) the rejected spelling is **`nameFilter`** (a natural
  casing/naming variant of the canonical `filter`), NOT the class-shaped
  `classFilter` of `#1` — so the friction here is *spelling-drift on the existing
  name-substring param*, the milder sibling of the missing-class-filter core. (b)
  The `UNKNOWN_PARAMS` message is now richer than `#1` recorded: it lists the full
  current param set `[filter, world, limit, fields, namesOnly]` AND tells the
  caller how to fetch the wiki page inline — which is why this was a clean
  one-retry. The `### actor.list` H3 added in `#3` (stating the narrowing param is
  `filter`, a name/label substring) is exactly the docs cue that would pre-empt a
  `nameFilter` first-guess, so this evidence supports that fix; an optional
  `nameFilter` alias on the slot (or an `UNKNOWN_PARAMS` suggestion for a
  `*[Ff]ilter`-ish unknown key) would let the natural guess self-correct without
  the round-trip. Friction note (verbatim): *"two parameter-name guesses were
  rejected (actor.list wants 'filter' not 'nameFilter'; actor.describe wants
  'actorName' not 'actor') but the error messages listed the valid keys so each was
  a single clean retry."* (The `actor.describe` half is tracked on
  `E-effect-actor-name-slot-vs-actorname #5`.)
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
