---
id: E-build-query-overlay-not-merged-into-served-page
title: "gameplay_tags.build_query served wiki page is a param-only stub — its rich op-token vocabulary is authored in a standalone wiki-src topic file that collides with the method slug and is silently shadowed, so it never reaches the served method page"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, wiki, gameplay-tags, build-query, discoverability, overlay-placement]
---

# `gameplay_tags.build_query` method page is a param-only stub — its overlay lives in a shadowed standalone topic file, not in the `### gameplay_tags.build_query` H3 section the renderer actually reads

The served `gameplay_tags.build_query` method page (reached via `call("gameplay_tags.build_query")`)
carries **only** the handler-registry summary line plus the four param rows:

```
# `gameplay_tags.build_query`
Namespace: `gameplay_tags`
Build an FGameplayTagQuery from a recursive JSON expression tree and optionally write it into a target asset property
**Parameters**
- `expression` (object, required): Root expression node: { op, tags?: string[], expressions?: [...] }
- `target` (object, optional): Optional { assetPath, propertyPath } write destination
- `maxDepth` (number, optional): Recursion depth guard for the expression tree Default: 12
- `description` (string, optional): User-provided description recorded on the FGameplayTagQuery
```

The load-bearing consequence is real: **the `op` token vocabulary is the central concept
of this method**, and the served page documents `expression.op` only as the free text
`{ op, tags?, expressions? }`. A caller cannot learn from the wiki that `op` must be one
of `any_tags_match` / `all_tags_match` / `no_tags_match` / `any_expressions_match` /
`all_expressions_match` / `no_expressions_match`, which take `tags` vs `expressions`, or
what fields come back (`tokenStreamBytes` / `wrote`). In the audited task the six op
names came from the task spec, not the wiki — an agent without that spec would guess or
read `GameplayTagBuildQueryHandler.cpp`.

## Root cause — overlay placed in the wrong file shape (NOT a generator merge bug)

The rich content WAS authored — as a **standalone** `docs/wiki-src/gameplay_tags.build_query.md`
topic file (op table, target-write CDO behavior, result shape, error codes, example).
But that is the wrong file shape for per-method overlay content, and it is dropped for
two compounding reasons, both verified against current source:

1. **Method pages read `### <ns>.<method>` H3 sections inside the NAMESPACE overlay, never
   standalone files.** `WikiHandler::RenderMethodPage` (`Catalog/WikiHandler.cpp:410-426`)
   appends `WikiOverlay::LoadMethodSection(Reg.MethodName)` as `## Notes`;
   `LoadMethodSection` (`Catalog/WikiOverlay.cpp:204-225`) strips the leaf to the Category,
   opens `gameplay_tags.md`, and returns the body of the `### gameplay_tags.build_query`
   H3 section. No such section existed (`gameplay_tags.md` had only `### gameplay_tags.list`),
   so the method page got nothing.
2. **The standalone topic file is permanently shadowed.** It is enrolled as a TopicNode
   (`WikiHandler.cpp:185-201`, which skips only Category-name slug collisions, not Method
   collisions), but `ClassifyNode` (`WikiHandler.cpp:207-240`) checks `MethodsByLowerName`
   FIRST, so `gameplay_tags.build_query` always classifies as a Method and the topic file
   is unreachable. The `## See also` link to it was therefore dead too.

This is **not** "a mechanical overlay-merge gap in the wiki generator" and **not**
systemic: the namespace-overlay sub-section mechanism WORKS — `### gameplay_tags.list`
already routes into the `gameplay_tags.list` method page as `## Notes` via the exact same
`LoadMethodSection` path (added by `E-gameplay-tags-list-default-limit-spills`). The
ticket's original "`### gameplay_tags.list` doesn't reach the per-method pages either" /
"`limit` default `500`" evidence was a **stale snapshot** (the live `gameplay_tags.list`
page carries the `### gameplay_tags.list` section and the clamped `default 50`); no
`wiki-generated/` tree is even present in the repo now. The defect is one mis-placed
overlay file, not a routing failure.

## Fix

Relocate the `gameplay_tags.build_query` overlay body from the standalone topic file into
a `### gameplay_tags.build_query` H3 section inside the namespace overlay
`docs/wiki-src/gameplay_tags.md` — exactly mirroring the sibling `### gameplay_tags.list`
section. The existing `LoadMethodSection` -> `RenderMethodPage` `## Notes` path then serves
the op-token table, result shape, error codes, and example on the
`call("gameplay_tags.build_query")` page with **no generator/code change**. Delete the now-redundant
standalone `gameplay_tags.build_query.md` (its slug-collision made it unreachable anyway)
and repoint the `## See also` bullet at the method page. (The broader "standalone wiki-src
file whose slug equals a registered method is silently shadowed" footgun is a separate,
optional guard — out of scope here.)

## Distinct from

- `B-wiki-namespace-underscore-not-found` (IN-REVIEW, High) — the **namespace index**
  page renders `# Not found:` for underscore namespaces (`call("gameplay_tags")` →
  not-found), a `NormalizeQuery` router bug. That ticket explicitly notes the
  **per-method** pages "all exist and are valid"; this ticket is that a per-method page,
  though it renders, has its **overlay body dropped**. Different surface (per-method page
  *content* vs namespace-index *resolution*).
- `E-wiki-page-filenames-dotted-flat-undiscoverable` (OPEN, Low) — the page **filename**
  layout is dotted-flat so direct nested-path Reads 404; that ticket states the content
  is "correct and complete once found." Here the content is *not* complete on the served
  page — the overlay body is missing — which is the opposite axis (content present-but-thin
  vs filename-discoverability).
- `F-gameplay-tag-query-authoring` / `F-gameplay-tags-namespace` (both DONE) — the feature
  tickets that *added* `build_query` and the registry. The method works (the audited task
  was clean); this is purely the served-doc discoverability of its op vocabulary.

## Evidence

Struggle-audit of a clean combat-prototype tag-taxonomy task (focus
`gameplay_tags.build_query`, namespace `gameplay_tags`, 12 calls, all `ok`, outcome
clean, judge filed nothing). Friction note, verbatim: *"the gameplay_tags.build_query
wiki page documents params/result only thinly and does NOT enumerate the op tokens
(all_expressions_match/any_tags_match/no_tags_match/all_tags_match) or the result fields
(tokenStreamBytes/wrote) -- its 'See also' promises a recursive expression-tree reference
that has no actual wiki page; the op names came from the task spec, not the wiki, so a
user without them would have to guess or read C++."*

Verified against the tree: `docs/wiki-src/gameplay_tags.build_query.md` was the rich
overlay (op table, result shape, error codes, worked example) — the very "recursive
expression-tree reference" the friction note says has no page. It was authored as a
**standalone topic file** whose slug equals the registered method `gameplay_tags.build_query`,
so `ClassifyNode` (method-before-topic) shadowed it and `RenderMethodPage` —
which only reads `### gameplay_tags.build_query` H3 sections out of `gameplay_tags.md` —
never saw it. From the runtime view the overlay's content was unreachable, which is why
the agent read it as "no actual wiki page." (The "every per-method `gameplay_tags.*` page
is a stub / `gameplay_tags.list` shows `limit` 500" claim was a stale pre-fix snapshot:
`### gameplay_tags.list` already routes into its method page and the live default is 50.)

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the combat-prototype `gameplay_tags.build_query` struggle audit (12 calls, all ok, outcome clean, judge filed nothing). PROCESS/docs friction: the served per-method wiki page is a param-only stub — `wiki-generated/gameplay_tags.build_query.md` carries only the handler-registry summary + four param rows, dropping the entire rich `docs/wiki-src/gameplay_tags.build_query.md` overlay (op-token table for all six ops, result shape `tokenStreamBytes`/`wrote`, error codes, worked example). The six `op` tokens — the method's core vocabulary — are therefore undiscoverable from the wiki; the audited agent got them from the task spec, and a caller without the spec would guess or read `GameplayTagBuildQueryHandler.cpp`. Confirmed systemic: every per-method `gameplay_tags.*` generated page is the same stub, and `gameplay_tags.list`'s generated page lacks the `### gameplay_tags.list` overview/`namesOnly` section the `E-gameplay-tags-list-default-limit-spills` fix added to the namespace overlay — so per-method overlays aren't merged and namespace-overlay sub-sections don't route to per-method pages. Dedup: ripgrep across OPEN/DONE/WONTFIX. `B-wiki-namespace-underscore-not-found` (IN-REVIEW) is the namespace-*index* not-found router bug and itself states per-method pages "all exist and are valid"; `E-wiki-page-filenames-dotted-flat-undiscoverable` (OPEN) is the dotted-flat *filename* 404 and states content is "complete once found" — neither owns the per-method overlay-body-dropped gap. `F-gameplay-tag-query-authoring`/`F-gameplay-tags-namespace` (DONE) are the feature tickets; the method works, this is its served-doc discoverability. Proposed fix: merge the matching `docs/wiki-src/<ns>.<method>.md` overlay body into the generated per-method page so `op` tokens + result fields are documented where the caller reads them; dev to judge whether the systemic fix is per-method-overlay merge, namespace sub-section routing, or both.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded: symptom CONFIRMED (the `call("gameplay_tags.build_query")` method page is genuinely summary + 4 params, op vocabulary absent) but the original "mechanical overlay-merge gap in the generator / systemic / `### gameplay_tags.list` doesn't route either / `limit` default 500" diagnosis was FALSE against current source. Verified: `RenderMethodPage` (`Catalog/WikiHandler.cpp:410-426`) serves per-method overlay content ONLY from `### <ns>.<method>` H3 sections inside the namespace file via `WikiOverlay::LoadMethodSection` (`Catalog/WikiOverlay.cpp:204-225`); `### gameplay_tags.list` already routes correctly through that path (proven by the IN-REVIEW `E-gameplay-tags-list-default-limit-spills` fix and the cloth/`asset.list` doc tests), and the live `gameplay_tags.list` default is 50. The real defect: the rich content was authored as a STANDALONE `docs/wiki-src/gameplay_tags.build_query.md` topic file whose slug equals the registered method, so `ClassifyNode` (method-before-topic, `WikiHandler.cpp:207-240`) permanently shadowed it and `LoadMethodSection` (which reads the `### gameplay_tags.build_query` H3 of `gameplay_tags.md`, absent) served nothing. Severity Medium->Low (docs/discoverability friction on a method the audited agent used 12/12 ok). Fix (zero code in the generator): moved the op-token table, target-write CDO note, result shape, error codes, and worked example into a new `### gameplay_tags.build_query` H3 section in `Docs/wiki-src/gameplay_tags.md`; lifted the `## See also` block into the namespace prelude (it was wrongly captured inside the `### gameplay_tags.list` section body, so it had been rendering on the list method page instead of the namespace page) and repointed its build_query bullet at `call("gameplay_tags.build_query")`; deleted the now-redundant, unreachable standalone `Docs/wiki-src/gameplay_tags.build_query.md`; switched the example's `"params"` key to the wire-correct `"args"`. Regression test: `Source/PinWright/Private/Tests/Infra/TestGameplayTagBuildQueryOverlayDocs.cpp` renders `gameplay_tags.build_query` through the live `WikiHandler::RenderPage` path and asserts the `## Notes` overlay markers (the six `op` tokens, `tokenStreamBytes`/`wrote` result fields, `UNKNOWN_OP`/`MIXED_PAYLOAD` error codes, `FGameplayTagQueryExpression`) — all overlay-exclusive, so reverting the H3 section makes `LoadMethodSection` return empty and the assertions fail. Files: `Docs/wiki-src/gameplay_tags.md`, `Docs/wiki-src/gameplay_tags.build_query.md` (deleted), `Source/PinWright/Private/Tests/Infra/TestGameplayTagBuildQueryOverlayDocs.cpp`.
