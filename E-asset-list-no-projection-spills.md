---
id: E-asset-list-no-projection-spills
title: "asset.list has no namesOnly/fields projection, so even a small pagination.limit page spills — the per-row tags array busts the inline budget on the enumerate-to-pick shape"
status: OPEN
severity: Low
category: ergonomic
tags: [asset, asset-list, response-size, oversized, projection, discovery]
encounters: 2
lastSeen: 2026-07-02T16:48:42.1520307+03:00
---

# `asset.list` has no per-row projection — the verbose `tags` array spills even a small page

`asset.list` is the canonical "what assets of class X / under path Y exist so I
can pick one?" discovery call. It **does** ship a working pagination lever
(`pagination.{offset,limit}`) — that half is a docs gap, owned by
`E-asset-list-pagination-undocumented-spills` (IN-REVIEW). This ticket is the
**other half**, a distinct code-side gap that the docs ticket explicitly flags as
out of scope: `asset.list` has **no `namesOnly`/`fields` per-row projection**, so
each returned row always carries `name`, `path`, `class`, `packagePath`, **and** a
verbose `tags` string array
(`Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp:866-883`). The
`tags` array is the heavy field, and there is no way to ask for just the
identity (`name`+`path`) the pick step actually needs.

The practical effect: pagination is **necessary but not sufficient**. Even a
deliberately small `pagination.limit:25` — the exact "keep it inline" remedy the
docs fix steers callers toward — still overflows on a populated content tree,
because 25 rows each carrying the full `tags` array cross the 10000-char inline
budget by ~4.8x. The response returns `outputTooLong` and the full payload is
written to `Saved/PinWright/HttpResponses/.../<uuid>.json`
(`E-http-response-spill`, DONE), forcing the caller to Read the spilled file (which
itself truncated at 420 of 846 lines) just to eyeball one static-mesh path. The
spill mechanism worked as designed; the friction is that the verb whose job is
"list assets so I can pick one" has no lever to drop the `tags` weight and stay
inline for the lightweight pick shape.

## What it should do

Give `asset.list` the same narrowing lever its sibling readers are getting: a
`namesOnly` boolean (or a `fields` projection / allow-list) that returns just
`name`+`path` per row and omits `class`/`packagePath`/the verbose `tags` array
unless asked. That keeps the textbook enumerate-to-pick call inline while the
full-fidelity row (with `tags`) stays available opt-in — mirroring the identical
lever already landed on `actor.list` (`E-actor-list-no-limit-spills`, `namesOnly`
+ `fields` allow-list) and proposed on `blueprint.list`
(`E-blueprint-list-no-projection-spills`) and `system.inspect.inspect_object`
(`E-inspect-object-no-projection-spills`).

**Docs page:** `docs/wiki-src/asset.md`, the existing `### asset.list` H3 (extend
it) — once the projection param lands, document that `namesOnly`/`fields` trims the
per-row payload for the "list so I can pick" case, so a small page no longer spills
on the `tags` weight.

## Distinct from

- `E-asset-list-pagination-undocumented-spills` (IN-REVIEW) — **same method,
  different fix family.** That ticket is docs-only: the `pagination.limit` lever
  already exists and works, and the fix is purely to document it (plus the
  top-level-`limit`-vs-`pagination.limit` UNKNOWN_PARAMS trap). Its own history #3
  explicitly notes that even a documented small `pagination.limit` STILL spills
  because the row is heavy, and flags a `fields`/`namesOnly` projection as "a
  possible future code-side follow-up (out of scope)." This ticket owns that
  code-side follow-up: the projection lever does **not** exist and adding it is a
  handler change, not a docs edit. Same lever-exists-vs-lever-missing split that
  `E-blueprint-list-no-projection-spills` already draws against the same docs ticket.
- `E-http-response-spill` (DONE) — the generic server-side spill mechanism itself;
  this ticket is that a specific verbose reader lacks a projection to stay under the
  threshold in the first place.
- `E-blueprint-list-no-projection-spills` (OPEN) / `E-inspect-object-no-projection-spills`
  (OPEN) / `E-get-nodes-pins-spill-no-projection` (IN-REVIEW) — identical
  projection-lever gap, different methods/handlers. Same proposed fix shape, same
  family; this is the `asset.list` member.
- `E-asset-list-path-ignored` (DONE) — same method, the top-level-`path`-silently-
  dropped bug; orthogonal, already fixed.

severity rationale: impact=pure-friction (a response spill that only forces a Read
+ the extra tax of a truncated spilled-file Read) × reach=asset.list is an
every-session discovery verb (which by the reach modifier would bump up), but held
at Low for consistency with the sibling projection-family tickets
(`E-blueprint-list-no-projection-spills`, `E-inspect-object-no-projection-spills`)
because strong workarounds remain — a very small `pagination.limit`, a tighter
`filter.class`/`path`, or `asset.search`/`asset.search_assets` — so the friction is
real but easily sidestepped -> Low

## Evidence

From the `landscape.create_grass_type` struggle audit (namespace `landscape`,
outcome **ergo** — the task otherwise ran clean and one-shot: the grass-type asset
was created, saved, and readback-verified exactly; the focus method itself had zero
friction). The friction was entirely in the mesh-discovery step. The caller's
corrected `asset.list {filter.class:/Script/Engine.StaticMesh, pagination:{limit:25}}`
— already the small-page "keep it inline" remedy — STILL returned
`{outputTooLong:true, message:"Response exceeds display limit (47761 chars,
threshold 10000); full payload written to ...HttpResponses/...json"}`. The agent
then had to Read the spilled **846-line** JSON — which itself truncated at line 420
("PARTIAL view — showing lines 1-420 of 846 total") — just to pick one static-mesh
path (`SM_Lightbulb`). CallAnalyzer SAY, verbatim: *"Even 25 rows overflowed due to
verbose tags."* Confirmed in source that the per-row `tags` array is the heavy
field with no projection to drop it (`AssetManageHandler.cpp:866-883`). This is the
process cost the pagination docs fix cannot remove: a small page still spills
because the row is heavy, so the projection is the actual remedy.

## History
- `#2-additional-spill-exceeds-read-cap` `OPEN` reporter — Additional evidence with a sharper angle on the process cost: a SECOND struggle-audit datapoint (focus `foliage.create_procedural`, namespace `foliage`, outcome **tool_bug** for the separate judge-filed `B-foliage-create-procedural-empty-callback-noop`; the `asset.list` friction is the distinct PROCESS angle). The caller's `asset.list {filter.class:StaticMesh, pagination.limit:40}` — the "list StaticMeshes so I can pick a plant stand-in" shape — returned `outputTooLong` at **76218 chars** (threshold 10000, ~7.6x over) and spilled to `Saved/PinWright/HttpResponses/.../<uuid>.json`. NEW ANGLE vs #1: whereas #1's 846-line spill only forced a single (truncated) Read, here the spilled JSON was **~32212 tokens — over the 25000-token Read cap — so a single Read was REJECTED**, forcing the agent into a 5-call Grep dance over the spill (grep `Rock|Plant|Bush`; grep `/Game` paths; grep `Rock|Cone|Sphere|Cube`; grep name fields; grep `totalCount`) just to extract ONE usable mesh path (`SM_Pillar_Debris_A`). SAY, verbatim: *"The tags make the payload heavy"* / *"The JSON is likely a single line."* So the missing `namesOnly`/`fields` projection escalates from "forces a Read" (#1) to "spilled file too big to Read at all → grep dance" — the strongest reach evidence yet that the per-row `tags` weight, not just the row count, is the driver. Same method, same root cause (no per-row projection lever; `AssetManageHandler.cpp:866-883` verbose `tags` array) as this ticket — a new occurrence, not a new file. Bumped `encounters`→2, `lastSeen`.
- `#1-initial-audit` `OPEN` reporter — Filed from the `landscape.create_grass_type` struggle audit (namespace `landscape`, outcome **ergo** — task otherwise clean/one-shot; the grass-type + landscape create/save/readback all round-tripped, focus method zero-friction). CallAnalyzer's second inefficiency (type F, "workaround") on `asset.list`, during mesh discovery: the corrected `asset.list {filter.class:/Script/Engine.StaticMesh, pagination:{limit:25}}` — already the small-page "keep it inline" remedy the sibling docs ticket steers toward — STILL overflowed at **47761 chars** (threshold 10000) and spilled to `Saved/PinWright/HttpResponses/.../<uuid>.json`, forcing a Read of the 846-line spilled JSON (which truncated at line 420) just to pick one static-mesh path. Root cause: each row carries a verbose `tags` string array (`AssetManageHandler.cpp:866-883`) and there is **no `namesOnly`/`fields` projection** to drop it. SAY, verbatim: *"Even 25 rows overflowed due to verbose tags."* Proposed: add a `namesOnly`/`fields` per-row projection (return just `name`+`path`, omit `class`/`packagePath`/`tags` unless asked), mirroring the landed lever on `actor.list` (`E-actor-list-no-limit-spills`) and the proposed one on `blueprint.list` (`E-blueprint-list-no-projection-spills`); document it in the existing `### asset.list` H3 of `docs/wiki-src/asset.md`. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX. The sibling `E-asset-list-pagination-undocumented-spills` (IN-REVIEW) is the **same method but a different fix family** — it is docs-only (the pagination lever exists) and its own #3 entry (which already captured THIS task's spill datapoint and bumped its own encounters→2) explicitly flags the `fields`/`namesOnly` projection as an out-of-scope future code-side follow-up; this ticket is that code-side follow-up, the same lever-exists-vs-missing split `E-blueprint-list-no-projection-spills` already draws against the docs ticket. `E-http-response-spill` (DONE) is the generic spill mechanism; `E-asset-list-path-ignored` (DONE) is the orthogonal top-level-path bug; `B-asset-list-short-class-ensure` is an unrelated class-ensure crash. No existing ticket owns the `asset.list`-has-no-projection code gap. Not re-filing the top-level-`limit` UNKNOWN_PARAMS half — that is fully owned by `E-asset-list-pagination-undocumented-spills` and this task's evidence for it is already appended there as #3.
