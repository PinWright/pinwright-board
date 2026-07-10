---
id: E-inspect-find-by-class-no-limit-spills
title: "system.inspect.find_by_class has no limit/projection — the very reader list_objects steers big-world callers to itself spills when a single class is populous (112 DynamicMeshActor rows), forcing a file Read to pick one representative"
status: OPEN
severity: Low
category: ergonomic
tags: [inspection, system-inspect, find_by_class, response-size, oversized, no-limit-spills, projection, pagination, docs]
encounters: 1
lastSeen: 2026-07-10T23:59:08.3929521+03:00
---

# `system.inspect.find_by_class` can't cap or project its rows — a populous class spills

`system.inspect.find_by_class` is the read-only "give me every actor of class X"
reader, and the method `system.inspect.list_objects` explicitly steers callers to
it for populated levels ("For large worlds, prefer system.inspect.find_by_class /
find_by_tag"). But `find_by_class` itself has **no narrowing lever**: its
registration declares only `className` (required) and `world` (default `auto`) —
**no `limit`, no `namesOnly`/`fields` projection, no cap** — then iterates every
matching actor and emits the whole `objects` array with `count` reflecting only
the returned set (no truncated total, because no cap exists).

So when the matched class is itself populous, the recommended "narrower" spills
just like the reader it was supposed to replace. On the Content Examples
`ExampleProjectWelcome`/`PersistentLevel` demo (227 actors), the single most-common
class is `DynamicMeshActor` (112 actors, ~half the level); `find_by_class
{className:"DynamicMeshActor"}` returns all 112 four-field rows, crosses the
10000-char display threshold, and spills to
`Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing a follow-up
filesystem Read.

## Why it matters (process cost in this task)

The task was a read-only composition audit whose final step was "a closer look at
**one representative** actor of the single most common class." The natural chain is
`list_actor_classes` (top class = DynamicMeshActor) then `find_by_class
{DynamicMeshActor}` to grab one example objectPath. But there is no `limit:1` (nor
any projection), so the caller must pull all 112 rows — which spills — then Read
the dumped file just to lift one representative path. One representative lookup
became 1 overflowing call + 1 file Read. A `limit:1` (or `namesOnly`) would have
kept the pick inline.

## What it should do

Mirror the narrowing levers already shipped/proposed on the sibling readers
(`E-inspect-list-objects-no-limit-spills`, `E-actor-list-no-limit-spills`), right-
sized to a flat enumeration (no pagination cursor):

- An optional `limit` (default `0` = all, so default output stays byte-identical)
  that caps rows after matching, while `count` reports returned rows, a new
  `totalCount` always reports the untruncated match count, and a `truncated` flag
  flips true when rows were elided — the same detectable-elision contract
  `system.inspect.list_actor_classes`/`list_objects` already use.
- A `namesOnly` / `fields` per-row projection (over `{name, label, path, class}`,
  dropping the verbose `path`) so a "just give me one/a few to pick from" survey
  stays inline — mirroring the `namesOnly`/`fields` the `list_objects` fix adds.
- Docs (`docs/wiki-src/system.inspect.md`, `### system.inspect.find_by_class`):
  note that matching a populous class exceeds the inline budget and spills, and
  that `limit`/`namesOnly` keep the result inline — so the "prefer find_by_class
  for large worlds" steer on `list_objects` doesn't just move the spill one method
  over.

## Repro (live, ExampleProjectWelcome / PersistentLevel, this HEAD)

Replayed via `mcp__pinwright__call`:

1. `system.inspect.list_actor_classes {}` → top class `{class:"DynamicMeshActor",
   count:112}` of 38 classes over 227 actors (inline; census reader has a `limit`).
2. `system.inspect.find_by_class {"className":"DynamicMeshActor"}` → 112 rows,
   payload over the 10000-char display threshold, spilled to a
   `Saved/EditorAutomation/HttpResponses/*.json` file; response has `count:112` but
   no `limit`/`truncated`/`totalCount` lever to have kept it inline.

For contrast, `system.inspect.find_by_class {"className":"PlayerStart"}` (count 1)
returns inline — the spill is purely a function of matched-row count, with no knob
to bound it.

Verbatim friction from the attempt: "two large RPC responses (find_by_class 112
rows, inspect_object) exceeding the 10k display threshold and spilling to disk,
requiring a follow-up file Read to extract the representative actor path."

Guilty source (no narrowing registered; uncapped emit), verified in synced source
`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp`:

- `:1527-1530` registers only `RPC_PARAM_REQ("className", ...)` +
  `RPC_PARAM_DEF("world", ..., "auto")` — no `limit`, no `fields`/`namesOnly`.
- `:1548-1556` `for (TActorIterator<AActor> It(World); It; ++It)` appends every
  match via `MakeInspectActorRow(Actor)` with no cap.
- `:1558-1564` emits the whole `objects` array + `count` (returned rows only), no
  `totalCount`/`truncated`.

## Distinct from

- `E-inspect-list-objects-no-limit-spills` (IN-REVIEW) — same shape (verbose reader,
  no limit/projection → spills) on the sibling `system.inspect.list_objects`, whose
  fix added `filter`/`limit`/`namesOnly`/`fields` to `list_objects` **only**;
  `find_by_class` is a separate registration in the same file that still has zero
  narrowing, and it is the exact method that ticket's `list_objects` doc steer
  points callers to. Not covered by that fix.
- `E-http-response-spill` (DONE) — the generic server-side file-reference spill
  mechanism; this ticket is that a specific reader has no way to stay under the
  threshold in the first place.
- `E-inspect-object-no-projection-spills` (WONTFIX) — same task's other spill on
  `inspect_object`, WONTFIX'd because `property.list`/`property.get` already offer
  an inline single-object read; that workaround does not apply to a multi-row class
  enumeration, which has no scoped alternative.
- `E-inspect-object-class-key-drift` / `E-inspect-find-by-tag-internal-name-not-label`
  — same/adjacent readers, unrelated angles (class-key naming; label omission).
- `B-actor-find-by-class-short-name-fails` — the mutating `actor.find_by_class`
  twin, a different bug (short-name resolution). Orthogonal.

severity rationale: impact=pure-friction (response spill only forces a Read) ×
reach=common reader but the overflow fires only when a single matched class is
populous -> Low

## History
- `#1-initial-repro` `OPEN` reporter — Filed from a realism-mode read-only
  composition-audit task on `ExampleProjectWelcome`/`PersistentLevel` (227 actors),
  outcome **done**, all calls `ok`/non-error. Replay-confirmed at HEAD:
  `system.inspect.find_by_class {className:"DynamicMeshActor"}` returns all 112 rows
  (the top class from `list_actor_classes`), overflows the 10000-char display
  threshold, and spills to a `Saved/EditorAutomation/HttpResponses/*.json` file,
  forcing a file Read just to pick one representative actor for the audit's final
  step — because `find_by_class` has no `limit` (not even `limit:1`) and no
  `namesOnly`/`fields` projection. `find_by_class {className:"PlayerStart"}` (count
  1) returns inline, confirming the spill is purely row-count driven with no knob to
  bound it. Grounded in synced source
  `Handlers/Environment/EnvironmentHandler.cpp:1527-1530` (only `className`+`world`
  registered), `:1548-1556` (uncapped `TActorIterator` append), `:1558-1564` (whole
  array emitted, `count` = returned rows only, no `totalCount`/`truncated`). Dedup:
  ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX for `find_by_class` and the
  no-limit-spills family — no ticket owns `find_by_class`'s missing narrowing;
  `E-inspect-list-objects-no-limit-spills` (IN-REVIEW) fixed only `list_objects` and
  its own doc steers callers to this method; `E-http-response-spill` (DONE) is the
  spill mechanism; `E-inspect-object-no-projection-spills` (WONTFIX) covers the
  single-object `inspect_object` spill (workaround `property.list`/`property.get`,
  which doesn't apply to a multi-row class enumeration). Proposed: add
  `limit`/`totalCount`/`truncated` + a `namesOnly`/`fields` projection mirroring the
  `list_objects`/`list_actor_classes` contract, plus a `docs/wiki-src/system.inspect.md`
  note so the "prefer find_by_class for large worlds" steer doesn't just relocate
  the spill.
