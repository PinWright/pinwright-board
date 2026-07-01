---
id: E-gameplay-tags-list-default-limit-spills
title: "gameplay_tags.list default limit (500) + fat per-row payload spills the un-prefixed overview to file on a normal (~400-tag) project"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [gameplay-tags, response-size, oversized, pagination, projection, docs]
---

# gameplay_tags.list spills to file on an unfiltered/overview listing

`gameplay_tags.list` is the obvious "what tags does this project already have?"
entry point, and the dominant reason to call it un-prefixed is to take a quick
**overview snapshot** of the registry (collision check, audit, "show me what's
there"). It already accepts narrowing/paging params (`prefix`, `source`,
`includeImplicitParents`, `limit`, `includeTotal` — `GameplayTagsHandler.cpp:321-327`),
so this is **not** a no-limit ticket. The gap is two-fold and lands the same
Read-tax the sibling verbose readers do:

1. **The default `limit` is 500** (`GameplayTagsHandler.cpp:326`,
   `const int32 Limit = FMath::Max(1, Ctx.GetInt("limit", 500))` :334). A normal
   project carries hundreds of tags — the `F-gameplay-tags-namespace` verify
   measured this very project at `totalMatches=401` (#5-verify-bounded-list). A
   plain `gameplay_tags.list` therefore serializes ~400 rows and blows past the
   **10000-char inline spill threshold**, returning `outputTooLong` and writing
   the full payload to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json` —
   forcing a Read/Grep of the spilled file just to eyeball what tags exist.

2. **Each row is fat.** `MakeTagRow` (`GameplayTagsHandler.cpp:68-88`) emits
   `name`, `source`, `comment`, `isExplicit`, **`sourceType`**, and a full
   **`configFile`** path *per tag*. The `configFile` absolute-ish path +
   `sourceType` string are pure overhead for the common "just show me the tag
   names" intent and dominate the bytes, so the payload crosses the threshold at
   a far smaller row count than the name-only data would.

The load-bearing defect is the **default sizing**: a bare `gameplay_tags.list`
serializes all ~400 fat rows and guarantees a spill. (Corroborating, not the
root cause: the friction call-log shows one overview call with an explicit
`limit=20` that was *also* reported written to file — `gameplay_tags.list`
`limit=20 (overall state; response written to file, too long)`. Whether 20 fat
rows cross the threshold depends on the per-tag `configFile` path / `comment`
lengths and the doubled content+structuredContent envelope, so that anecdote is
borderline supporting evidence, not the primary defect.) The prefix-filtered
`gameplay_tags.list` calls in the same task — `prefix=Ability` and
`prefix=Ability.Combat source=CombatAbilities` — returned inline fine; only the
un-prefixed overview overflowed, which is the precise shape this ticket is
about.

## What it should do

Mirror the fixes already shipped/proposed for the sibling verbose readers
(`E-recorder-list-sessions-limit`, `E-graph-connections-pagination`,
`E-actor-list-no-limit-spills`, `E-skeleton-list-bones-no-limit-spills`,
`E-volume-get-info-no-limit-spills`):

- Lower the **default `limit`** to a value that stays inline on a normal project
  (the current 500 guarantees a spill at 400+ tags), keeping `totalMatches`/
  `totalMatchesExact`/`truncated` (already emitted, :413-415) so elision stays
  detectable. The bounded-scan machinery is already in place — only the default
  is mis-sized for the inline budget.
- Add a `namesOnly` / `fields` projection so the dominant "just list the tag
  names" case drops the per-row `configFile` + `sourceType` (and optionally
  `comment`), shrinking each row to a fraction of its size and letting a much
  larger listing stay inline.
- **Docs (`docs/wiki-src/gameplay_tags.md`):** in the `### gameplay_tags.list`
  section note that an un-prefixed listing returns *all* tags, that on a normal
  project (hundreds of tags) the result exceeds the inline budget and spills to
  file, and that `prefix`/`source` (or a smaller `limit` / the proposed
  `namesOnly`) are the way to keep an overview inline.

## Distinct from

- `E-http-response-spill` (DONE) — the *generic* server-side spill mechanism
  (the file-reference fallback itself); this ticket is that a *specific verbose
  reader* overflows by default, the same relationship the no-limit-spills family
  has to the spill mechanism.
- `E-actor-list-no-limit-spills` / `E-skeleton-list-bones-no-limit-spills` /
  `E-volume-get-info-no-limit-spills` (OPEN) — identical *shape* (verbose reader
  → spill → forced Read) but those readers have **no** `limit` at all; this one
  *has* a `limit` whose **default (500) is sized above the inline budget** plus a
  fat per-row payload, so the un-prefixed overview spills by default. Same
  proposed fix family (lower default + `namesOnly`/`fields` projection),
  different RPC and different root cause within the family.
- `F-gameplay-tags-namespace` (DONE) — the feature ticket that *added*
  `gameplay_tags.list` with bounded scanning; the listing works correctly, this
  is purely the response-size / default-sizing ergonomics on top of it.

## Evidence

From the combat-ability tag-taxonomy task (focus `gameplay_tags.remove`,
namespace `gameplay_tags`, outcome `clean`, `filed_id` empty — the judge filed
nothing and the self-reported friction note was *"none — wiki pages documented
every param and MCP responses were self-describing"*). This response-size /
Read-tax friction is a distinct PROCESS angle the friction note did not call
out: in a 16-call task the single un-prefixed overview listing —
`gameplay_tags.list` `limit=20 (overall state; response written to file, too
long)` — spilled to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`,
forcing a follow-up Read just to see the project's existing tags, while every
prefix-narrowed `gameplay_tags.list` call in the same task returned inline.
Project scale corroborated by `F-gameplay-tags-namespace #5-verify-bounded-list`
(`totalMatches=401` on this same host), against a default `limit` of 500
(`GameplayTagsHandler.cpp:326/334`) and a per-row payload that includes the full
`configFile` path + `sourceType` (`GameplayTagsHandler.cpp:81-84`).

## History
- `#2-reword-and-fix` `IN-REVIEW` developer — Disposition REWORD then implemented as GO. The adversarial lens objected (the headline over-stated the single `limit=20` anecdote as the primary defect); reworded the title/body/Distinct-from to demote the `limit=20` call-log to corroborating evidence and frame the load-bearing defect as default-500 + fat-rows sizing (the other two lenses voted valid and verified every cited line against current source). Fix mirrors the accepted sibling family (`E-actor-list-no-limit-spills`, `E-console-search-default-limit-spills`): lowered `gameplay_tags.list` default `limit` 500→50 and clamped to `[1, 500]` (`GameplayTagsHandler.cpp` registration + `FMath::Clamp(Ctx.GetInt("limit", 50), 1, 500)`); added a per-row `fields`/`namesOnly` projection via the shared `FHandlerContext::ReadFieldProjection` (namesOnly → name+source+isExplicit, dropping the byte-dominating `configFile` path + `sourceType` + `comment`) threaded through a new `FTagRowFields` gate into `MakeTagRow`; `totalMatches`/`totalMatchesExact`/`truncated` unchanged. Docs: added a `### gameplay_tags.list` section to `docs/wiki-src/gameplay_tags.md` documenting the overview-overflows-inline behavior and the `prefix`/`source`/`limit`/`namesOnly`/`fields` levers. Files: `Source/PinWright/Private/Handlers/GameplayTags/GameplayTagsHandler.cpp`, `docs/wiki-src/gameplay_tags.md`. Regression test: `PinWright.gameplay_tags.ListFieldProjection` (`Source/PinWright/Private/Tests/Gameplay/TestGameplayTagsHandlers.cpp`) seeds a tag in a known source and asserts the unprojected row keeps every column (name/source/comment/isExplicit/sourceType/configFile), `namesOnly:true` drops configFile/sourceType/comment while keeping name/source/isExplicit, `fields:["name"]` emits only `name`, and an over-large `limit` clamps to ≤500 rows — fails if the projection wiring or the lowered/clamped default is reverted.
- `#1-initial-audit` `OPEN` reporter — Filed from the combat-ability tag-taxonomy struggle audit (focus `gameplay_tags.remove`, outcome clean, judge filed nothing, self-reported friction "none"). Distinct PROCESS angle: the un-prefixed overview `gameplay_tags.list` call overflowed the 10000-char inline budget and spilled to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing a Read — call-log verbatim `gameplay_tags.list limit=20 (overall state; response written to file, too long)`, while the prefix-narrowed list calls returned inline. Root cause: default `limit=500` (`GameplayTagsHandler.cpp:326/334`) on a ~400-tag project (`F-gameplay-tags-namespace #5` measured `totalMatches=401` on this host) plus a fat per-row payload that emits `configFile` + `sourceType` per tag (`MakeTagRow` :81-84). Proposed: lower the inline default `limit`, add a `namesOnly`/`fields` projection to drop the `configFile`/`sourceType` columns, and document the overview-overflows-inline behavior in the `### gameplay_tags.list` section of `docs/wiki-src/gameplay_tags.md`. Dedup: ripgrep across OPEN/DONE/WONTFIX — `F-gameplay-tags-namespace`/`F-gameplay-tag-query-authoring` (both DONE) are the feature tickets that added the namespace, no size-ergonomics angle; `E-actor-list-no-limit-spills`/`E-skeleton-list-bones-no-limit-spills`/`E-volume-get-info-no-limit-spills` (OPEN) are the same shape but on readers with *no* limit param at all (this one has a limit whose default is mis-sized). No existing ticket owns the `gameplay_tags.list` response-size gap.
