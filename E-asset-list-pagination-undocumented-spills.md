---
id: E-asset-list-pagination-undocumented-spills
title: "asset.list has a working pagination.limit but the wiki never mentions it (while its two sibling list verbs document their limit), so the natural call goes unpaginated and spills to the HttpResponses file"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [asset, response-size, oversized, pagination, docs]
encounters: 2
lastSeen: 2026-07-02T14:57:46.2335539+03:00
---

# asset.list's pagination lever is undocumented, so the obvious call spills

`asset.list` is the obvious "what assets of class X / under path Y exist?" entry
point. Unlike `actor.list` / `system.inspect.list_objects` (the
`*-no-limit-spills` family), it is **not** missing a cap — the handler
(`Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp:688-695`)
already registers a `pagination` object whose `offset`/`limit` are honored
(`:728-736`, `:842-857`), and the response already emits `totalCount` (full
match count), `count` (returned rows), and `offset`
(`:891-897`) — the same detectable-elision contract the spill-family tickets ask
other readers to grow. The code is fine.

The gap is **discoverability**. The `### asset.list` overlay in
`docs/wiki-src/asset.md` (lines 162-173) documents only the `filter.class`
short-name/full-path table and the top-level-`path` honoring — it says **nothing**
about `pagination.limit`/`offset` or the `totalCount`/`count` readback. The two
sibling list verbs **immediately below it in the same overlay** each document
their cap explicitly: `asset.search` — *"`limit` (default 50, max 500)"* (line
179); `asset.search_assets` — *"`limit` (default 100, `0` = unlimited)"* (line
185). So a caller reading the overlay sees two of the three list verbs advertise a
`limit` and the third (`asset.list`) advertise none — and `asset.list`'s cap is
not even a top-level `limit` but is nested one level down under `pagination`, the
least guessable of the three shapes. The natural call therefore goes unpaginated.

On a populated content tree the unpaginated per-asset array — each row carries
`name`, `path`, `class`, `packagePath`, **and** a `tags` string array
(`:866-883`) — crosses the 10000-char inline budget, so the response returns
`outputTooLong` and the full payload is written to
`Saved/EditorAutomation/HttpResponses/.../<uuid>.json` (`E-http-response-spill`,
DONE), forcing the caller to Read the spilled file just to pick one asset. The
spill mechanism worked exactly as designed; the friction is that nothing in the
docs steers the caller to the `pagination.limit` that already exists and would
have kept the listing inline.

## What it should do

This is a **docs-only** fix (the code already has the lever) —
`docs/wiki-src/asset.md`, the existing `### asset.list` H3 (extend it, do not
create it):

- Document the `pagination` object (`{ offset, limit }`) and the response
  `totalCount` / `count` / `offset` fields, the way the sibling
  `asset.search` / `asset.search_assets` sections already document their `limit`.
- Note that an unpaginated class/path list overflows the inline budget on any
  populated tree and spills to the HttpResponses file, and that passing
  `pagination.limit` (and reading `totalCount` to detect elision) keeps the
  listing inline — so callers reach for it up front.
- Optionally note the per-row shape includes the verbose `tags` array, so a
  large-`limit` page is heavier per row than the bare name/class the common
  "list so I can pick" intent needs (a future `fields`/`namesOnly` projection
  would be the code-side follow-up, but that is out of scope for this docs gap).

## Distinct from

- `E-actor-list-no-limit-spills` / `E-inspect-list-objects-no-limit-spills`
  (IN-REVIEW) and the rest of the `*-no-limit-spills` family — those readers
  **genuinely lack any cap** and the fix is to *add* `limit`+projection to the
  handler. `asset.list` is the opposite: the cap already ships and works; the
  fix is purely to *document* it. Different fix family (docs vs. code), so a
  separate ticket.
- `E-asset-list-path-ignored` (DONE) — same method, the top-level-`path`-silently-
  dropped bug; orthogonal gap, already fixed.
- `E-http-response-spill` (DONE) — the generic server-side spill mechanism
  itself; this ticket is that a specific verbose reader's existing narrowing
  lever is undocumented, so the caller never reaches for it.

## Evidence

From the catalog-hero asset-preview struggle audit (focus
`render.capture_asset_preview`, namespace `render`, outcome **clean** — all 11
calls `ok`/non-error; the three captures round-tripped at exact requested
dimensions). The task's friction note, verbatim: *"the only minor detour was
asset.list exceeding the display limit so its payload was read from the spilled
HttpResponses JSON file, which was straightforward."* The story's step 1 was the
textbook "list so I can pick" shape — *"list the static meshes available in the
project content (asset.list with filter.class = StaticMesh, …) and pick a concrete
one."* So even with the intent fully satisfied by one pick, the un-paginated
`asset.list` (call #6, `args_summary:"path=/Engine/BasicShapes
filter.class=StaticMesh"`) crossed the inline budget and forced a Read of the
spilled file — when a `pagination.limit` the docs never surfaced would have kept it
inline. The reporter labels it "straightforward," which is exactly the
normalization the docs should pre-empt by surfacing the lever.

## History
- `#2-docs-pagination` `IN-REVIEW` developer — Docs-only fix as scoped: extended the existing `### asset.list` H3 in `Docs/wiki-src/asset.md` (the param names `pagination`/`offset`/`limit` already auto-render from the AssetManageHandler.cpp:693 registration, so the H3 now adds the parts the gloss lacks). Documented (1) the `pagination` object `{ offset, limit }` with its default `-1`=unlimited and that the cap is nested under `pagination` rather than a top-level `limit` like the sibling search verbs; (2) the `totalCount`/`count`/`offset` readback contract — `count < totalCount` is the detectable-elision signal; (3) that the unpaginated unlimited default overflows the 10000-char inline budget on any populated tree (each row = name+path+class+packagePath+verbose `tags` array) and spills to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, with the steer to pass `pagination.limit` up front to keep the listing inline; (4) a note that the heavy per-row shape favors a small `limit` for "list so I can pick" and that a `fields`/`namesOnly` projection is a possible future code-side follow-up (out of scope). No code change — the lever already ships and works. Regression test: `Source/PinWright/Private/Tests/Infra/TestAssetListPaginationDocs.cpp` (`FAssetListPaginationDocTest`) renders the live `asset.list` method page through `WikiHandler::RenderPage` (production doc-render path, not a copy of the overlay) and asserts on overlay-exclusive markers — the `totalCount` response field (not a param, never auto-rendered), the spill/HttpResponses note, and the keep-it-`inline` guidance — all of which vanish when the overlay H3 is reverted. Adversarial-lens caveat noted: the ticket's `/Engine/BasicShapes` repro (~6 meshes) is too small to actually spill, so it was NOT reused as a demonstration; the general gap (unpaginated `asset.list` on a populated tree spills + the lever was undocumented) is independently real and is what the fix addresses. Files: `Docs/wiki-src/asset.md`, `Source/PinWright/Private/Tests/Infra/TestAssetListPaginationDocs.cpp`.
- `#1-initial-audit` `OPEN` reporter — Filed from the catalog-hero asset-preview struggle audit (focus `render.capture_asset_preview`, namespace `render`, outcome **clean** — all 11 calls `ok`/non-error). Friction note, verbatim: *"the only minor detour was asset.list exceeding the display limit so its payload was read from the spilled HttpResponses JSON file, which was straightforward."* Verified in source that this is a **docs gap, not a code gap**: `asset.list` (`Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp:688-695`) already registers a `pagination` object honored at `:728-736`/`:842-857`, and emits `totalCount`/`count`/`offset` at `:891-897` — but the `### asset.list` overlay in `docs/wiki-src/asset.md` (lines 162-173) documents only `filter.class` + top-level `path`, never the `pagination.limit`/`offset` lever or the `totalCount`/`count` readback, while the two sibling list verbs right below it (`asset.search` line 179, `asset.search_assets` line 185) each document their `limit`. So the natural unpaginated call (each row = name+path+class+packagePath+tags array, `:866-883`) overflows the 10000-char budget and spills to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json` (`E-http-response-spill`, DONE), forcing a Read just to pick one mesh. Proposed: extend the existing `### asset.list` H3 in `docs/wiki-src/asset.md` to document `pagination.{offset,limit}` + the `totalCount`/`count`/`offset` readback and note that an unpaginated list spills / that `pagination.limit` keeps it inline — mirroring the sibling sections. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — the `*-no-limit-spills` family (`E-actor-list-no-limit-spills`, `E-inspect-list-objects-no-limit-spills`, `E-volume-get-info-no-limit-spills`, `E-skeleton-list-bones-no-limit-spills`, etc.) is the opposite case (readers that genuinely lack a cap → code fix to add one); `asset.list` already has the cap, so this is a docs-only ticket. `E-asset-list-path-ignored` (DONE) is the orthogonal top-level-path bug; `E-http-response-spill` (DONE) is the spill mechanism. No existing ticket owns the `asset.list` undocumented-pagination/spill-discoverability gap.
- `#3-additional-small-page-still-spills` `OPEN` reporter — Additional evidence: a THIRD struggle-audit datapoint (focus `landscape.create_grass_type`, namespace `landscape`, outcome **ergo** — task otherwise clean/one-shot; the grass-type + landscape create/save/readback all round-tripped). Reproduces BOTH halves of this gap in one story and adds a sharper angle on the spill half. (a) Explicit half: the caller's natural first `asset.list filter.class=StaticMesh limit=40` passed a top-level `limit` (mirroring the sibling search verbs) and got the same hard reject verbatim — `[UNKNOWN_PARAMS] Unknown parameter(s) for 'asset.list': [limit]. Valid parameters: [path, filter, recursive, pagination, depth]. Call 'asset.list' with no 'args' field to fetch its wiki page.` — then wiki-nav'd and corrected to `pagination.limit`. Replay-confirmed at HEAD via `mcp__pinwright__call asset.list {filter.class:StaticMesh, limit:40}` → identical UNKNOWN_PARAMS. (b) NEW ANGLE on the spill half: the corrected call was NOT the unpaginated default this ticket's #1 describes — it already passed `pagination.limit=25`, the exact "keep it inline" remedy the docs fix steers callers toward — and it STILL overflowed. Replay-confirmed at HEAD via `mcp__pinwright__call asset.list {filter.class:/Script/Engine.StaticMesh, pagination:{limit:25}}` → `outputTooLong` at **47761 chars** (threshold 10000), spilled to `Saved/PinWright/HttpResponses/.../<uuid>.json`, forcing an extra Read to pick a mesh. So on a populated tree even a modest 25-row page busts the budget by ~4.8x because each row still carries the verbose `tags` array — i.e. the documented `pagination.limit` steer is necessary but NOT sufficient; a caller must pick a very small limit (or reach for the still-hypothetical `fields`/`namesOnly` projection this ticket already flags as the code-side follow-up). Strengthens the case that the per-row `tags` weight, not just the unlimited default, is what forces the spill. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — same method, same undocumented-pagination + heavy-row root cause as this ticket; a new occurrence, not a new file. Bumped `encounters`→2, `lastSeen`.
- `#2-evidence-top-level-limit-rejected` `OPEN` reporter — Second struggle-audit datapoint confirming the predicted misuse-then-correct cycle from the undocumented nesting. Focus `animation.authoring.add_state_machine` (namespace `animation.authoring`, outcome **ergo** — task otherwise clean, 27/28 calls ok). Step 1 was the textbook "list so I can pick a skeleton" intent: `asset.list filter.class=Skeleton`. The caller's natural first attempt passed a **top-level** `limit` (call #11) — exactly the shape the two sibling verbs `asset.search`/`asset.search_assets` document — and got the hard reject `[UNKNOWN_PARAMS] Unknown parameter(s) for 'asset.list': [limit]. Valid parameters: [path, filter, recursive, pagination, depth]. Call 'asset.list' with no 'args' field to fetch its wiki page.`; call #12 then succeeded by nesting it under `pagination.{offset,limit}`. Friction note, verbatim: *"first asset.list failed with [UNKNOWN_PARAMS] because limit must go inside the nested pagination object, not top-level (fixed via the wiki page)."* This is the active half of the same docs gap: #1-initial-audit caught the **silent** consequence (a no-`limit` call spilling to HttpResponses); this catches the **explicit** consequence (guessing `limit` top-level → UNKNOWN_PARAMS → wiki-nav → corrected to `pagination.limit`), a wasted call plus a forced wiki read. Strengthens the proposed `### asset.list` overlay fix in `docs/wiki-src/asset.md`: documenting that `limit`/`offset` live **under `pagination`** (not top-level like the siblings) pre-empts both the spill and this UNKNOWN_PARAMS round-trip. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — same method, same undocumented-pagination root cause as this ticket; not a new ticket. `B-unknown-params-error-suggests-deleted-question-mark-suffix` (DONE) owns the *error-string wording*, not the param-shape discoverability, so this stays here.
