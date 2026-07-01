---
id: E-actor-find-by-name-substring-key-self-contradicts
title: "actor.find_by_name's required key is 'name' but its own param help calls the value a 'Substring' — callers guess key 'substring' and eat MISSING_REQUIRED_PARAM 'name'"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [actor, docs, find_by_name, param-name, substring, self-contradicting-help, discovery]
encounters: 1
lastSeen: 2026-06-24T19:46:41Z
---

# `actor.find_by_name` describes its value as a "Substring" but keys it `name` — the help word becomes the wrong-guess param key

`actor.find_by_name`'s sole required param is keyed **`name`**, but both the
method summary and the param help describe the *value* as a **substring**, not a
name:

- Method help (`Handlers/Actor/QueryHandler.cpp:165`): *"Search the current level
  for actors whose label, name, or path contains the given **substring**
  (case-insensitive)."*
- Param help (`:167`): `RPC_PARAM_REQ("name", "string", "**Substring** to match
  against label/name/path; rejected if it contains '..', '/', or '\\' as a
  path-traversal guard.")`.

So a caller who has just read (or recalls) that the verb takes "a substring"
reaches for the key **`substring`** — the exact noun the help uses — and the
dispatcher rejects it with `[MISSING_REQUIRED_PARAM] Missing required parameter
'name' (type: string)`. The value really *is* a substring (it's matched with
`Contains`), so the help is accurate about semantics; it's the **key spelling**
(`name`) that the help never surfaces, and the word it *does* surface
(`substring`) is the most natural wrong guess. This is a self-contradicting-help
ergonomic, not a missing-alias path-key case.

It is also the **mirror image** of the `E-effect-actor-name-slot-vs-actorname`
family: there the canonical key is `actorName` and callers wrongly guess `name`
(because `name` is the search key on `find_by_name`); *here* the canonical key
**is** `name` and callers wrongly guess `substring`. Both are first-guess
discoverability round-trips, but the rejected key and the affected verb are
different, so this is not covered by that ticket (which explicitly relies on
`name` being the *correct* `find_by_name` key, `QueryHandler.cpp:167`, to argue
`name` should stay unaliased on the `actorName` slots).

## Repro

From the campfire-clearing layout-preview struggle audit (namespace `effect`,
outcome `ergo`; the judge filed the OUTCOME defect
`E-effect-list-debug-shapes-types-not-drawn` for the unrelated
`list_debug_shapes` can't-confirm-drawn-shapes gap). At the light-verification
step the agent guessed the key `substring` **twice** before reading the wiki and
switching to `name`:

1. `actor.find_by_name {substring:"Campfire_Light"}`
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'name' (type: string)`
2. `actor.find_by_name {substring:"Moonlight_Fill"}`
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'name' (type: string)`
3. (wiki-nav) → `actor.find_by_name {name:"Campfire_Light"}` → success
   (`/Script/Engine.PointLight`)
4. `actor.find_by_name {name:"Moonlight_Fill"}` → success
   (`/Script/Engine.SpotLight`)

Two wasted `is_error` calls + one wiki read, zero blocked progress. Friction note
(verbatim): *"Minor: actor.find_by_name rejected my guessed param 'substring'
twice before I read its wiki page and used 'name'."* The two-in-a-row repeat is
notable — the caller didn't learn from the first rejection because the error
names the *missing* key (`name`) but not that `substring` was the offending guess,
so the second call repeated the same wrong key on the next light before the wiki
disambiguated it.

## What it should do

**Fix (docs-first — the established `actor.*` first-guess-param-key remedy).** This
friction class is resolved on the board the same way the sibling `actor.list`
wrong-key ergonomic was (`E-actor-list-no-class-filter #3-docs-fix`): a docs/help
reword + an `### actor.<verb>` H3 in the existing `docs/wiki-src/actor.md` overlay +
a `WikiHandler::RenderPage` regression test — **no dispatcher alias** (the
mirror `E-effect-actor-name-slot-vs-actorname #4`/`#5` deliberately declined to add
human-shorthand aliases to actor-identity slots, citing the DONE precedent
`E-material-editor-param-name-drift #2`). Two edits, both rendering onto the
`actor.find_by_name` method page:

- **Param/method help reword (`Handlers/Actor/QueryHandler.cpp:165`/`:167`):** lead
  the param help with the key so the read-the-help path no longer plants `substring`
  as the obvious key — e.g. the `name` description now reads *"Search substring
  matched case-insensitively against label/name/path. The param key is 'name' (the
  value is a substring, but the key is not 'substring'); …"*, and the method summary
  names the `name` param instead of describing a bare "Substring".
- **`### actor.find_by_name` H3 in `docs/wiki-src/actor.md`:** add the per-verb H3
  (the overlay has `### actor.list`/`### actor.describe`/etc. but **none for
  `### actor.find_by_name`** today) stating the search substring goes in the key
  **`name`** (NOT `substring`/`query`/`search`), that it's a case-insensitive
  `Contains` over label/name/path with the `..`/`/`/`\` traversal guard, and that the
  rejection names the missing `name` rather than the offending guess.

**Alias deliberately declined (was the original lead proposal).** Adding `substring`
(±`query`/`search`) as an alias on the `name` slot would also close the round-trip
and `substring` is non-polysemous on this verb, but it runs against the established
`actor.*` direction above (docs-overlay edit IS the fix; aliases are declined), so it
is out of scope here.

## Not a duplicate of

- `E-effect-actor-name-slot-vs-actorname` (IN-REVIEW) — the inverse direction
  (canonical key `actorName`, wrong guess `name`/`label`/`actor`); that ticket
  depends on `name` being the *right* key for `find_by_name`, which is exactly the
  key this ticket says callers fail to guess.
- `E-actor-list-no-class-filter` (IN-REVIEW) — `actor.list`'s `filter` slot and
  class-vs-name narrowing; a different verb and a different rejected key
  (`classFilter`/`nameFilter`), not `find_by_name` + `substring`.
- `E-effect-list-debug-shapes-types-not-drawn` (judge-filed this task) — the
  OUTCOME gap (`list_debug_shapes` returns only the static supported-type set, so
  it can't confirm drawn shapes); unrelated to this `find_by_name` param-key
  friction.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the campfire-clearing
  layout-preview struggle audit (namespace `effect`, outcome `ergo`; judge filed
  the unrelated OUTCOME ticket `E-effect-list-debug-shapes-types-not-drawn`).
  Distinct PROCESS angle: `actor.find_by_name` keys its required param `name`, but
  the method help AND param help both call the value a "**Substring**"
  (`QueryHandler.cpp:165`/`:167`), so the agent guessed key `substring` **twice**
  (`{substring:"Campfire_Light"}` then `{substring:"Moonlight_Fill"}`, both
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'name'`) before a wiki-nav
  fixed it to `{name:…}` (both succeeded). Two wasted `is_error` calls + one wiki
  read. Root cause is self-contradicting help: the help teaches the noun
  `substring` as the value while the key is `name`, and the error names the
  missing key but not the offending guess, so the wrong key repeated across two
  lights. Mirror image of `E-effect-actor-name-slot-vs-actorname` (which relies on
  `name` being the correct `find_by_name` key) and distinct from
  `E-actor-list-no-class-filter` (different verb/key). Proposed: add a `substring`
  (±`query`/`search`) alias on the `name` slot of `actor.find_by_name`, OR reword
  the param help to lead with the key, plus an `### actor.find_by_name` H3 on
  `docs/wiki-src/actor.md` (none exists today; `### actor.list`/`### actor.describe` do).
- `#2-reword` `OPEN` lead-engineer — Redirected the fix from the dispatcher-alias
  lead to the board's established `actor.*` first-guess-param-key remedy (docs-first,
  no alias), matching `E-actor-list-no-class-filter #3-docs-fix` and the alias-declined
  mirror `E-effect-actor-name-slot-vs-actorname #4`/`#5` (cite DONE precedent
  `E-material-editor-param-name-drift #2`). Every factual claim in the body verified
  against current source (`QueryHandler.cpp:165`/`:167`/`:170`; no `### actor.find_by_name`
  H3 in `docs/wiki-src/actor.md`); severity Low / category ergonomic unchanged.
  Demoted the `substring`(±`query`/`search`) alias to an explicitly-declined
  alternative and made the wiki H3 part of THIS ticket's fix (the body had framed it
  as "downstream wiki process, not this ticket").
- `#3-docs-fix` `IN-REVIEW` developer — Implemented the reworded docs-first fix.
  (1) Reworded the `actor.find_by_name` method summary and the `name` param help in
  `Source/PinWright/Private/Handlers/Actor/QueryHandler.cpp:165`/`:167` to lead with
  the key (*"The param key is 'name' (the value is a substring, but the key is not
  'substring')"*) — no behavior change, the handler still reads `Ctx.GetString("name")`.
  (2) Added an `### actor.find_by_name` H3 to the existing overlay
  `Docs/wiki-src/actor.md` stating the substring goes in the `name` key (NOT
  `substring`/`query`/`search`), the case-insensitive `Contains` over label/name/path,
  the `..`/`/`/`\` traversal guard, and that the rejection names the missing `name`
  rather than the offending guess. Regression test:
  `FWikiHandlerFindByNameDocumentsNameKeyTest` in
  `Source/PinWright/Private/Tests/Infra/TestWikiHandler.cpp` renders
  `actor.find_by_name` via `WikiHandler::RenderPage` and asserts on two
  one-edit-each markers — `param key is 'name'` (the param-help reword, in the auto
  Parameters list) and `most natural wrong guess` (overlay-exclusive H3 prose) — so it
  fails iff either half of the fix is reverted. No dispatcher alias added, matching the
  sibling `actor.*` pattern.
