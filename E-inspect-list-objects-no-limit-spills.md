---
id: E-inspect-list-objects-no-limit-spills
title: "system.inspect.list_objects has no filter/limit/projection at all — on any populated level it dumps every actor and spills to the HttpResponses file, forcing a Read just to eyeball the scene"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [inspection, response-size, oversized, pagination, projection, docs]
---

# system.inspect.list_objects has no narrowing — dumps every actor and spills

`system.inspect.list_objects` is the read-only "what objects are in this world?"
entry point and the natural first call when surveying a level. But the handler
(`EnvironmentHandler.cpp:1359-1388`) registers **only** a `world` param —
**no `filter`, no `limit`, no field projection** — then loops
`TActorIterator<AActor>` (:1370) appending `name`/`path`/`class` per actor with
**no cap** (:1374-1377) and emits the whole `objects` array (:1381). `count` is
reported (:1382) but only reflects the returned set; there is no truncated total
because no cap exists.

This is even **less narrowable than its mutating twin `actor.list`** (which at
least ships a substring `filter` param): `list_objects` exposes no narrowing lever
whatsoever short of switching to a different method. The registration doc itself
concedes the problem — *"For large worlds, prefer system.inspect.find_by_class /
find_by_tag"* — i.e. the method's own documentation steers callers away from it on
exactly the populated levels where the "show me everything" intent is most
natural.

On any populated level the per-actor row count alone pushes the serialized array
past the 10000-char spill threshold (`E-http-response-spill`, DONE), so the
response returns `outputTooLong` and the full payload is written to
`Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing the caller to
**Read the spilled file** just to eyeball the scene. The spill mechanism itself
works correctly; the gap is that this verbose reader has no way to stay under the
threshold in the first place.

## What it should do

Mirror the narrowing levers already shipped on sibling readers (and proposed for
`actor.list` in `E-actor-list-no-limit-spills`), right-sized to the evidence — a
flat actor enumeration, so no pagination cursor:

- An optional `filter` (substring on name/class) so callers can scope the same way
  `actor.list` already can — its absence here is a gratuitous asymmetry with the
  mutating twin the doc explicitly calls a "counterpart" of.
- An optional `limit` (default `0` = all, so default output stays byte-identical)
  that truncates after filtering while `count` reports returned rows, a new
  `totalCount` always reports the untruncated match count, and a `truncated` flag
  flips true when rows were elided — the same detectable-elision contract
  `E-recorder-list-sessions-limit` and `E-graph-connections-pagination` landed.
- A `namesOnly` / `fields` per-row projection so the common "just give me names so
  I can pick" case drops the per-actor `path` (the longest field) — mirroring the
  `fields` allow-list `actor.describe` already ships.
- **Docs (`docs/wiki-src/system.inspect.md` overlay, `### system.inspect.list_objects`
  section):** note that the method returns *all* actors, exceeds the inline budget
  on any populated level, and that `filter` / `limit` / `namesOnly` / `fields` keep
  a listing inline — superseding the current "prefer find_by_class/find_by_tag"
  steer, which is a workaround for the missing levers, not a fix.

## Evidence

From the system health-check / inventory struggle audit (focus `null`, namespace
`system`, outcome **clean** — every one of the 13 calls `ok`/non-error). Step 2 of
the story was the textbook "enumerate so I can pick" shape: *"List the actors in
the level so I can see what's there, then narrow in on the lights."* The audited
call is `system.inspect.list_objects` `{world:"editor"}` (call #4 of 13,
`args_summary:"world=editor (spilled to file)"`) on the Content Examples
`ExampleProjectWelcome`/`PersistentLevel` demo level (227 actors).

Friction note, verbatim: *"Mostly smooth; only minor friction was two large
responses (list_objects 92KB, inspect_object 59KB) spilling to disk files that I
had to read via the filesystem instead of inline."* So the un-narrowed
`list_objects` produced a **92KB** payload that crossed the 10000-char threshold,
spilled to the HttpResponses file, and was read off disk — when the actual intent
("see what's there, then narrow to lights") was immediately satisfied by the three
`find_by_class` calls that followed (PointLight→7, DirectionalLight→1,
SpotLight→0). A `filter`/`namesOnly` projection would have kept the survey inline.

## Distinct from

- `E-http-response-spill` (DONE) — the *generic* server-side spill mechanism (the
  file-reference fallback itself); this ticket is that a *specific verbose reader*
  has no narrowing to stay under the threshold, the same relationship
  `E-recorder-list-sessions-limit`, `E-graph-connections-pagination`,
  `E-volume-get-info-no-limit-spills`, `E-skeleton-list-bones-no-limit-spills`, and
  `E-actor-list-no-limit-spills` all have to the spill mechanism.
- `E-actor-list-no-limit-spills` (IN-REVIEW) — same *shape* (no limit/projection →
  spill → forced Read) but on the **mutating** `actor.list` in a **different
  handler/file** (`Actor/QueryHandler.cpp`); that one already ships a substring
  `filter`, whereas `list_objects` ships none. Same fix family, different RPC and
  source file; this ticket additionally asks for the missing `filter` param to
  reach parity with the twin the doc calls a "counterpart."
- `B-inspect-object-omits-component-properties` (DONE) — the *other* large response
  in the same task's friction note (`inspect_object` at 59KB). That 59KB is the
  *intended* result of that ticket's fix (it now emits a full reflected `properties`
  dump, 120-138KB on real targets); it is a separate method with no missing
  narrowing gap, so it is **not** re-filed here. Only the `list_objects` 92KB
  reflects an absent-narrowing defect.
- `B-inspect-misses-pie-world` (references `list_objects`) — a *world-resolution*
  bug (queries miss the active PIE world), an orthogonal gap on the same method.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the system health-check / inventory struggle audit (focus `null`, namespace `system`, outcome **clean** — all 13 calls `ok`/non-error; the seed verbs round-tripped). `system.inspect.list_objects {world:"editor"}` (call #4 of 13) on the Content Examples `ExampleProjectWelcome`/`PersistentLevel` level (227 actors) returned a **92KB** payload that crossed the 10000-char spill threshold and was written to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing a filesystem Read just to survey the scene — when the intent ("see what's there, then narrow to lights") was satisfied by the three `find_by_class` calls that followed. Verified in source: the handler (`EnvironmentHandler.cpp:1359-1388`) registers ONLY `world` (:1361) — no `filter`/`limit`/projection — loops every `TActorIterator<AActor>` (:1370), appends name/path/class with no cap (:1374-1377), emits the whole array (:1381); even less narrowable than `actor.list`, which at least ships `filter`. Its own registration doc concedes the problem ("For large worlds, prefer find_by_class/find_by_tag"). Friction note, verbatim: *"only minor friction was two large responses (list_objects 92KB, inspect_object 59KB) spilling to disk files that I had to read via the filesystem instead of inline."* Proposed: add `filter` (parity with the `actor.list` twin), `limit` (default `0`=all + untruncated `totalCount` + `truncated`, per `E-recorder-list-sessions-limit`/`E-graph-connections-pagination`), a `namesOnly`/`fields` projection (per `actor.describe`), and a `### system.inspect.list_objects` note in `docs/wiki-src/system.inspect.md`. Dedup: ripgrep across OPEN/DONE/WONTFIX — `E-actor-list-no-limit-spills` (IN-REVIEW) is the same shape on the mutating `actor.list` in a different file (already has `filter`); `E-http-response-spill` (DONE) is the spill mechanism; `E-volume-get-info-no-limit-spills` / `E-skeleton-list-bones-no-limit-spills` (OPEN) are the same shape on other readers; `B-inspect-object-omits-component-properties` (DONE) owns the *other* large response in this task's note (`inspect_object` 59KB is its fix's intended output, not a narrowing defect — not re-filed); `B-inspect-misses-pie-world` is an orthogonal world-resolution bug on this method. No existing ticket owns the `system.inspect.list_objects`-has-no-narrowing gap.
- `#2-narrowing-levers` `IN-REVIEW` developer — Added the missing narrowing levers to `system.inspect.list_objects`, mirroring the proven `actor.list` (`Actor/QueryHandler.cpp`) pattern and reusing the shared `FHandlerContext::ReadFieldProjection` helper. In `Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp` (handler at the top of the `system.inspect.list_objects` block) the registration now declares `filter` (substring on actor name+class), `limit` (default `0`=all), `fields`, and `namesOnly` alongside the existing `world`; the body filters, caps after filtering, and emits `count` (returned rows) + `totalCount` (full untruncated match count) + `truncated` (rows elided) — the same detectable-elision contract as `actor.list`/`E-graph-connections-pagination`. Default output stays byte-compatible (no filter, `limit=0` → all rows, all three `name`/`path`/`class` keys, plus the new always-present `totalCount`/`truncated`). The `namesOnly` column set is `{name, class}` (drops the verbose `path`); `fields` is the case-insensitive allow-list over `{name, path, class}`. Updated the registration Summary to advertise the new knobs and dropped the "prefer find_by_class/find_by_tag for large worlds" steer. Docs: added a `### system.inspect.list_objects` H3 section to `docs/wiki-src/system.inspect.md` documenting `filter`/`limit`/`totalCount`/`truncated`/`namesOnly`/`fields` and noting they supersede the old find_by_class/find_by_tag steer. Regression test: `Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp` → `PinWright.system.inspect.list_objects.LimitAndProjection` spawns three real PointLights (guarded by `FScopedEditorWorldActorGuard`), filters to them, and asserts the uncapped baseline (count==totalCount==3, truncated:false), `limit=2` (count 2, totalCount 3, truncated:true, objects length 2), `namesOnly` (row keeps name+class, drops path), and `fields=[class]` (row keeps only class) — all of which fail if the levers were reverted. Did not compile/run (a later phase does).
