---
id: E-actor-list-no-limit-spills
title: "actor.list has no limit or field projection — a normal populated level (227 actors) overflows the 10k spill threshold and forces a Read of the HttpResponses file just to pick a handful of actors"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [actor, response-size, oversized, pagination, projection, docs]
---

# actor.list has no limit/projection — it dumps every actor and spills to file

`actor.list` is the obvious "what actors are in this level?" entry point, and the
overwhelmingly common reason to call it is to **pick a few concrete actors to act
on** (feature in a sequence, parent something to, animate, etc.). But the method
enumerates **every** `AActor` in the resolved world and returns the full array in
one payload, with only one *substring* narrowing param (`filter` = name/label) and
**no `limit` and no field projection**
(`QueryHandler.cpp:34-77` registers only `filter` + `world`; the body loops
`TActorIterator<AActor>` :48 and unconditionally appends `label`/`name`/`path`/
`class` per actor :57-64 with no cap, then emits the whole `actors` array :69).
`count` is already reported (:70) but only reflects the returned set — there is no
untruncated total because no cap exists yet.

On any populated level the per-actor row count alone pushes the serialized array
past the 10000-char spill threshold, so the response comes back as
`outputTooLong` and the full payload is written to
`Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing the caller to
**Read/Grep the spilled file** just to eyeball a handful of names. The spill
mechanism itself works correctly (`E-http-response-spill`, DONE); the gap is that
this verbose reader has no narrowing to stay under the threshold in the first
place, so even the simplest "show me what's here so I can pick a few" call
overflows on a normal level. Other tickets corroborate the scale of the dump on
populated worlds: `B-export-snapshot-empty-stub` measured an `actor.list` at
**121,942 chars**, and `B-get-components-renders-empty-to-caller #5` / 
`E-actor-describe-no-header-only-read` both cite an `actor.list` at ~18,310 chars
— routinely 2x-12x over the budget, never anything pathological.

The substring `filter` helps only if the caller *already knows* a discriminating
name fragment. The natural "just show me everything so I can choose" call — the
exact shape of the task that surfaced this — is the one that overflows, and the
caller can't ask for "just the first N" or "just names+class".

## What it should do

Right-sized to the evidence (which only ever wanted label/class to confirm/pick,
never `path`), mirroring the narrowing levers already shipped on sibling readers —
**no cursor needed** (this is a flat list, not a paged graph):

- An optional `limit` (default `0` = all, so the default output stays byte-identical
  for callers that don't pass it) that truncates after filtering while `count`
  reports the returned rows, a new `totalCount` always reports the untruncated
  match count, and a `truncated` flag flips true when rows were elided — the same
  detectable-elision contract `E-recorder-list-sessions-limit` (limit default 20 /
  `0`=all + `totalCount`) and `E-graph-connections-pagination` (`truncated` +
  `totalMatched`) landed.
- A `namesOnly` / `fields` per-row projection so the common "just give me the
  labels so I can pick" case drops the per-actor `path` (the longest field) and
  returns a tiny payload — mirroring the `fields` allow-list `actor.describe`
  already ships (`E-actor-describe-no-header-only-read #5`).
- **Docs (`docs/wiki-src/actor.md`):** the `### actor.list` overlay section
  **already exists** (landed by `E-actor-list-no-class-filter #3`, documenting only
  the name-vs-class-filter confusion). **Extend** it — do not create it — with a
  note that the method returns *all* actors, that the result exceeds the inline
  budget on any populated level, and that `limit` / `namesOnly` / `fields` (or
  `filter`) keep a listing inline.

**Fix:** Add `limit` (default `0`=all; emit `totalCount` + `truncated` alongside the
existing `count`) and a `namesOnly`/`fields` per-row projection (allow-list of
`label`/`name`/`path`/`class`, defaulting to all four so unprojected output is
unchanged) to the `actor.list` handler in `QueryHandler.cpp`, and extend the
existing `### actor.list` section of `docs/wiki-src/actor.md` with the spill/limit
note. No pagination cursor — a flat actor list does not need one.

## Evidence

From the establishing-shot cinematic task (focus `sequencer.add_actors`, namespace
`sequencer`, outcome `ergo`; the seed `add_actors` itself round-tripped clean — 5
actors bound + 5 transform tracks verified). Call-log, verbatim: `actor.list`
`{world:"editor"}` → *"227 actors, payload spilled to disk"*. The task's intent at
that step was tiny — the story says *"find some actors that are already placed …
pick a handful of concrete ones to feature"* — yet the only way to enumerate them
was to pull all 227 rows, which crossed the 10000-char threshold and spilled to
`Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, requiring a follow-up Read
of the file just to choose 5 actors. (The task's named friction note is about a
separate neighbor verb, `sequencer.add_camera`, filed as
`E-sequencer-add-camera-no-actor-path`; this response-size friction on `actor.list`
is a distinct process angle the friction note did not call out.)

## Distinct from

- `E-http-response-spill` (DONE) — the *generic* server-side spill mechanism (the
  file-reference fallback itself); this ticket is that a *specific verbose reader*
  has no narrowing to stay under the threshold in the first place, the same
  relationship `E-recorder-list-sessions-limit`, `E-graph-connections-pagination`,
  `E-volume-get-info-no-limit-spills`, and `E-skeleton-list-bones-no-limit-spills`
  all have to the spill mechanism.
- `E-actor-list-no-class-filter` (OPEN) — same method, **different gap**: that is
  the *class-filter param-name* confusion (a class-intent caller guesses
  `classFilter`, eats `UNKNOWN_PARAMS`, falls back to a fragile name-substring) and
  its fix is a docs cross-reference to `actor.find_by_class`. This ticket is the
  *absence of a size cap / projection* so even a correct, narrowing-free call
  overflows and forces a Read. They share the proposed `docs/wiki-src/actor.md`
  overlay target but are different gaps with different fix families.
- `E-volume-get-info-no-limit-spills` / `E-skeleton-list-bones-no-limit-spills`
  (OPEN) — identical *shape* (no limit/projection → spill → forced Read) but on
  `volume.get_volumes_info` and `skeleton.list_bones`, different methods/handlers.
  Same proposed fix family, different RPC.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the establishing-shot cinematic struggle audit (focus `sequencer.add_actors`, outcome ergo; the seed verb round-tripped clean). `actor.list` has no `limit`/pagination/projection (`QueryHandler.cpp:34-77` registers only `filter`/`world`; loops all `TActorIterator<AActor>` :48, serializing label/name/path/class per actor :57-64 with no cap, emits whole array :69). Call-log: `actor.list {world:"editor"}` → "227 actors, payload spilled to disk" — the response crossed the 10000-char spill threshold and was written to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing a Read of the spilled file just to pick a handful of actors to feature, which was the whole point of the call. Corroborated by other tickets measuring `actor.list` at 121,942 chars (`B-export-snapshot-empty-stub`) and ~18,310 chars (`B-get-components-renders-empty-to-caller #5`, `E-actor-describe-no-header-only-read`). Proposed: add `limit` (default-inline, 0=all, with untruncated `count`) per `E-recorder-list-sessions-limit`/`E-graph-connections-pagination`, optionally a `fields`/`namesOnly` projection, and document the all-actors-overflows-inline behavior in the `### actor.list` section of `docs/wiki-src/actor.md`. Dedup: ripgrep across OPEN/DONE/WONTFIX — every other ticket referencing `actor.list` cites it only incidentally; `E-actor-list-no-class-filter` (OPEN) is the same method but the class-filter param-name gap (different fix family); `E-http-response-spill` (DONE) is the spill mechanism; `E-volume-get-info-no-limit-spills` + `E-skeleton-list-bones-no-limit-spills` (OPEN) are the same shape on other readers. No existing ticket owns the `actor.list`-has-no-limit/projection gap.
- `#2-recurrence-staging-pass` `OPEN` reporter — Recurrence from the lighting/staging-pass struggle audit (focus `actor.get`, namespace `actor`, outcome clean; every call `ok`/non-error — the seed verbs all round-tripped). The task's opening intent was the textbook "just show me everything so I can choose" shape this ticket calls out: story says *"list the actors in the active world so I can see what's there. Pick one concrete existing actor from that list."* Friction note, verbatim: *"only minor handling was actor.list exceeding the display limit (auto-written to disk) and the dump being too large for a single Read, so I grepped/offset-read it to pick an actor."* So even on this open level the un-narrowed `actor.list {}` (call #2 of 9, `args_summary:"{}"`) overflowed the inline budget, spilled to the HttpResponses file, and then was *too large for one Read* — forcing grep + offset-read just to select a single actor (DirectionalLight_0). This is the second independent task where the natural "enumerate so I can pick" call forces a spilled-file Read; the added wrinkle here is that the spilled dump itself exceeded a single Read's capacity, so the caller paid a grep-then-offset-read tax on top of the spill. Reinforces the proposed `limit` + `fields`/`namesOnly` projection (a `namesOnly` projection would have kept this inline) and the `docs/wiki-src/actor.md` `### actor.list` note. No new ticket filed — same gap, same method, same fix family.
- `#3-recurrence-gearroom-cutscene` `OPEN` reporter — Third independent recurrence, from the GearRoom-cutscene scaffolding struggle audit (focus `sequencer.list_track_types`, namespace `sequencer`, outcome clean — all 13 calls `ok`/non-error). Same textbook "enumerate so I can pick" shape: step 1 of the story is *"list the actors in the level and pick two existing ones … Confirm they actually exist before using them."* The audited call is `actor.list` `{}` (call #2 of 13, `args_summary:"{} list all level actors"`) on the Content Examples `ExampleProjectWelcome`/`ExampleProjectWelcome`-style demo level. Friction note, verbatim: *"actor.list overflowed the 10k display threshold and spilled to a file, but that is expected behavior and grep on the spill file resolved all three actors cleanly."* The reporter explicitly labels the spill "expected behavior" — which is exactly the normalization this ticket pushes back on: a `{}` enumeration whose sole purpose was to confirm three named actors (BP_Gears_146 / StaticMeshActor_1 / DirectionalLight_0) exist should not have to cross the inline budget and force a grep of the HttpResponses spill file. Reinforces the proposed `limit` (default-inline, `0`=all, untruncated `count`) and, more pointedly here, a `fields`/`namesOnly` projection — the caller needed only label/class to confirm existence, never `path` (the longest per-row field). The `docs/wiki-src/actor.md` `### actor.list` note (already landed via `E-actor-list-no-class-filter #3`) would also set the expectation that a bare `actor.list` overflows on any populated level so callers reach for narrowing up front. No new ticket filed — same gap, same method, same fix family.
- `#4-reword-and-fix` `IN-REVIEW` developer — Validity review (correctness/adversarial/historian) confirmed the defect is real and present in current source (`QueryHandler.cpp` actor.list still registers only `filter`+`world`, loops every actor, appends 4 fields with no cap) and distinct from `E-http-response-spill` (DONE) and `E-actor-list-no-class-filter` (IN-REVIEW). Reworded two stale framings the lenses flagged: (a) dropped the "no pagination cursor" ask — a flat actor list needs `limit` + projection, not a cursor (gold-plating vs. the evidence, which only ever wanted label/class); (b) corrected the docs ask from "create the `### actor.list` section" to "extend" it — that H3 already exists (added by `E-actor-list-no-class-filter #3`, `docs/wiki-src/actor.md`), and only the spill/limit note was missing. Implemented the fix: `actor.list` now takes `limit` (default `0`=all; emits the existing `count` plus a new untruncated `totalCount` and a `truncated` flag — same detectable-elision contract as `recorder.list_sessions` / `graph.connections`) and a `namesOnly`/`fields` per-row projection (allow-list of `label`/`name`/`path`/`class`, default keeps all four so unprojected output is byte-identical; mirrors `actor.describe`'s `fields` reader). Extended the existing `### actor.list` overlay with the spill note + the three narrowing levers. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Actor/QueryHandler.cpp` (handler), `docs/wiki-src/actor.md` (docs). Regression test `FActorListLimitAndProjectionTest` (`actor.list.LimitAndProjection`) added to `Source/EditorAutomationRpcGateway/Private/Tests/World/TestActorHandlers.cpp`: spawns 3 real PointLights into the editor world, filters to them, asserts `limit=2` caps rows at 2 while `totalCount`=3 / `truncated`=true, and that `namesOnly`/`fields=["label"]` drop the `path` row field — all against the production handler, so reverting the limit/projection plumbing fails it.
- `#5-recurrence-possess-roundtrip` `IN-REVIEW` reporter — Fourth independent recurrence, from the PIE-possession-roundtrip struggle audit (focus `editor.possess`, namespace `editor`, outcome `tool_bug` — the judge filed `B-editor-possess-non-pawn-silent-success` for a *separate* defect; this response-size friction is a distinct PROCESS angle the judge ticket did not own). Same textbook "enumerate so I can pick" shape: story step 2 is *"Discover what's actually in the current level: call actor.list to enumerate the actors … and pick a concrete, real spawnable pawn/character or notable static actor by its actual label … use one that genuinely exists in this host level."* The audited call is `actor.list` `{}` (call #2 of 13, `args_summary:"{} (payload spilled to disk, 119k chars)"`) on the Content Examples `ExampleProjectWelcome` level. Friction note, verbatim: *"actor.list response (119k chars) overflowed the display limit and spilled to a disk file I had to Read/Grep through to find a target."* This pins a concrete **119,000-char** measurement on the spill, corroborating the 121,942-char figure already cited from `B-export-snapshot-empty-stub` — the un-narrowed `actor.list {}` on a populated ContentExamples level routinely lands ~12x over the 10000-char inline budget. The caller needed only a single real label to possess (it picked `SK_DinoDragon`), never `path` — the exact case the proposed `namesOnly`/`fields` projection (and `limit`) would have kept inline, sparing the Read/Grep tax. No new ticket filed — same gap, same method (`actor.list`), same fix family already landed in `#4`; this is cross-task evidence aggregation onto the owning ticket.
- `#6-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
