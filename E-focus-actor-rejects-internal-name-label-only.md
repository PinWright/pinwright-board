---
id: E-focus-actor-rejects-internal-name-label-only
title: "editor.focus_actor matches ONLY the display label and rejects the internal object name with [ACTOR_NOT_FOUND] — the inverse of the actor.* verbs, surfaced as a 'not found' error for an actor that demonstrably exists"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [editor, focus_actor, actorname, display-label, internal-name, actor-not-found, cross-method-consistency, docs]
---

# `editor.focus_actor` accepts only the display label, not the internal object name — opposite of the `actor.*` verbs, and the error reads as "actor doesn't exist"

`editor.focus_actor` resolves its `actorName` slot by matching **only** the
actor's display label, case-insensitively:

```cpp
// ViewportHandler.cpp:147
if (Actor->GetActorLabel().Equals(ActorName, ESearchCase::IgnoreCase)) { ... }
// no match → ViewportHandler.cpp:158
Ctx.SendError(TEXT("ACTOR_NOT_FOUND"), TEXT("Actor not found"));
```

The param help (`ViewportHandler.cpp:132`) is honest that it wants a label
("Display label of the actor to focus on; case-insensitive match against actor
labels"), **but that label-only contract is the inverse of every per-actor
`actor.*` verb**, which accept *either* the display label *or* the unique
internal object name (see `E-actor-name-resolution-label-collision`, where the
internal name is the documented collision-safe key). So a caller who has just
run `actor.list` — whose entries are keyed by the **internal name**
(`BP_Gears_146`) — naturally feeds that exact `name` to `focus_actor` and gets
`[ACTOR_NOT_FOUND]`. The error text says the actor was *not found*, which flatly
contradicts the real cause: the actor exists and was just enumerated; only its
*internal name* (not its label `BP_Gears`) was passed. The caller cannot tell
from the error whether the actor is missing or whether they used the wrong
identifier kind, so the failure mode is a confusing-error + cross-method
inconsistency, not a clean rejection.

This is distinct from the existing label-resolution tickets:
- `E-actor-name-resolution-label-collision` — the **input-side discoverability**
  gap on the `actor.*` verbs: those verbs *do* accept the internal name, and the
  ticket asks the docs to flag it as the collision-safe disambiguator. Here the
  problem is the opposite: `focus_actor` *refuses* the internal name.
- `E-inspect-find-by-tag-internal-name-not-label` — the read-twin/write-twin
  `name`-field meaning asymmetry on `find_by_tag`. Unrelated to `focus_actor`'s
  resolution rule.
- `E-actor-verbs-reject-actorpath-slot` / `E-actor-select-singular-actorname-rejected`
  — input-key drift and singular-vs-array arity; neither touches the
  label-only-vs-name-too acceptance split that `focus_actor` exhibits.

## What it should do

Pick ONE and document it; today the overlay does neither:

1. **Preferred — match the `actor.*` verbs:** have `editor.focus_actor` resolve
   `actorName` against the internal object name (the object-path leaf returned in
   `actor.list` / `actor.find_by_class`) **as well as** the display label, so the
   identifier the caller already has from `actor.list` just works. This removes
   the inverse-of-`actor.*` surprise entirely.
2. **If label-only is intentional:** make the error self-explaining instead of a
   bare "Actor not found" — e.g. `[ACTOR_NOT_FOUND] No actor with display label
   '<X>' (this method matches labels, not internal object names; pass the label
   from actor.get / actor.find_by_class)`, and document on the overlay that
   `focus_actor` is label-keyed while the `actor.*` verbs also accept the
   internal name.

Either way, add a `### editor.focus_actor` section to
`docs/wiki-src/editor.md` (which today only lists `editor.focus_actor` once in
the Viewport bullet at line 16 and has **no** per-method section for it),
stating which identifier(s) `actorName` accepts and pointing at how to obtain a
label vs. an internal name.

## Evidence (this task — gears beauty-shot, seed `editor.set_game_view`)

15-call workflow, OUTCOME clean. The `focus_actor` step took three calls:

| # | call | result |
|---|------|--------|
| 3 | `editor.focus_actor` (wiki-nav) | ok (doc read) |
| 4 | `editor.focus_actor actorName=BP_Gears_146` | **is_error** `[ACTOR_NOT_FOUND] Actor not found` |
| 5 | `editor.focus_actor actorName=BP_Gears` (the label) | ok |

`actor.list` had just confirmed the actor existed under the internal name
`BP_Gears_146` (the story explicitly names it `"BP_Gears_146"` and asks to
confirm via `actor.list`). Friction note, verbatim: *"Minor: editor.focus_actor
rejected the exact internal name \"BP_Gears_146\" with [ACTOR_NOT_FOUND] because
it matches the actor LABEL (\"BP_Gears\") per its wiki, not the actor.list
\"name\" field; one extra retry plus checking the dump to confirm uniqueness
before passing the label. Everything else was smooth."*

So the caller burned a wiki read, one failed call, and a dump re-check to learn
that the identifier `actor.list` reports is *not* the one `focus_actor` accepts —
an inverse-of-`actor.*` rule the error message hides behind "Actor not found".

**Workaround:** Pass the actor's **display label** (e.g. `BP_Gears`) to
`editor.focus_actor`, not the internal `name` from `actor.list`; obtain the
label from `actor.get` / `actor.find_by_class` (their `name`/label fields) before
calling.

## History
- `#2-focus-actor-shared-resolver` `IN-REVIEW` developer — Adopted option 1 (the "Preferred" fix): `editor.focus_actor` now resolves `actorName` through the shared `McpActorUtils::FindActorByName` (label OR internal object name OR object path) instead of its inline `GetActorLabel()`-only loop, so the internal `name` field `actor.list` / `actor.find_by_class` report (`BP_Gears_146`) now focuses directly — matching every `actor.*` verb. Also rewrote the `[ACTOR_NOT_FOUND]` message to name the identifier kinds tried instead of a bare "Actor not found", and updated the `actorName` param help. Files: `Source/PinWright/Private/Handlers/Editor/ViewportHandler.cpp` (handler body + param help); `Docs/wiki-src/editor.md` (added a `### editor.focus_actor` per-method section stating which identifiers `actorName` accepts). Regression test: `Source/PinWright/Private/Tests/EditorOps/TestEditorHandlers.cpp` → `PinWright.editor.focus_actor.ResolvesInternalName` spawns a real actor with a display label deliberately distinct from its internal object name, invokes the production handler with the INTERNAL name, and asserts success + that the actor became selected; reverting to label-only matching fails it (internal name never equals the distinct label).
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a gears beauty-shot task (seed `editor.set_game_view`; 15 calls, OUTCOME clean, PROCESS friction only). `editor.focus_actor actorName=BP_Gears_146` (the internal name `actor.list` reports, and the exact name in the story) failed `[ACTOR_NOT_FOUND]`; only `actorName=BP_Gears` (the display label) succeeded. `ViewportHandler.cpp:147` matches `GetActorLabel()` exclusively, the inverse of the `actor.*` verbs (which accept label OR internal name per `E-actor-name-resolution-label-collision`), and the "Actor not found" text contradicts the real cause (the actor exists; wrong identifier kind). Dedup (ripgrep over OPEN+closed; qmd unavailable): no existing `focus_actor` ticket; not covered by `E-actor-name-resolution-label-collision` (that is the actor.* input-side label-collision discoverability gap, opposite acceptance rule), `E-inspect-find-by-tag-internal-name-not-label` (read/write twin name-field asymmetry), `E-actor-verbs-reject-actorpath-slot`, or `E-actor-select-singular-actorname-rejected`. Proposed: accept the internal object name as well as the label (match the `actor.*` verbs), OR make the error self-explaining and add a `### editor.focus_actor` section to `docs/wiki-src/editor.md` (no per-method section exists today) stating which identifier `actorName` accepts.
