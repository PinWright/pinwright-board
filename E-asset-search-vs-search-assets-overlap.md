---
id: E-asset-search-vs-search-assets-overlap
title: "Overlapping asset list/search verbs (asset.list vs asset.search vs asset.search_assets, plus a phantom asset.find) — the asset overlay never reconciles them, so an agent picks the wrong one for a class-only list"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, asset, search, search_assets, misuse-then-correct, discovery, wiki]
---

# `asset.search` and `asset.search_assets` overlap in name and intent but split required params — and the `asset.md` overlay documents neither

The `asset` namespace ships **two** verbs that both answer "find assets", with
near-synonymous names but different required-parameter contracts:

- **`asset.search`** (`AssetManageHandler.cpp:945`, "Search for assets **by
  name** using substring or wildcard matching") — **requires `query`**
  (a name pattern); optional `path` / `classFilter` / `classPathFilter` /
  `parentClassPath` / `classFilterMode` / `limit`. Empty `query` returns
  `MISSING_REQUIRED_PARAM` / `INVALID_ARGUMENT`.
- **`asset.search_assets`** (`AssetQueryHandler.cpp:166`, "Search for assets
  **using filters**") — **all params optional**; class-driven listing via
  `classNames[]` / `packagePaths[]` / `recursivePaths` / `recursiveClasses` /
  `limit`. No name `query` at all.

So the two verbs partition the search space by *which axis is primary*:
`asset.search` is name-first (and needs a name), `asset.search_assets` is
class/path-first (and needs no name). For the very common "list all assets of
class X" intent (no name pattern in hand), `asset.search_assets { classNames }`
is the right verb and `asset.search` hard-errors on the missing `query`. The
names give no hint of this split — `asset.search` reads like the more general /
default verb, so an agent reaching for "search for SoundWaves" naturally tries
it first and eats the missing-param error before discovering the `_assets`
sibling is the class-list verb.

The discoverability gap is concrete: the `asset` overlay
(`docs/wiki-src/asset.md`) documents `asset.list`, `asset.dump`,
`asset.dump_folder`, `asset.map_references`, and `asset.fixup_redirectors` — but
**neither `asset.search` nor `asset.search_assets` appears by name**, and there
is no "which search verb when" note. An agent navigating the overlay for an
asset-listing recipe finds no guidance on the two-verb split, so the wrong-verb
attempt is essentially unavoidable for a class-only list.

This is the misuse-then-correct friction shape: a misleading/under-specified
method name leads to one wrong call, then a corrected variant. It is the
asset-namespace twin of the param-name-drift cluster
(`E-asset-path-vs-assetpath-list-drift`, `E-class-name-format-inconsistency`):
adjacent verbs in one namespace disagree on their contract and the overlay
never reconciles them, so the caller pays a trial-and-error round-trip.

## Evidence (this task)

`audio.authoring` SoundClass-tree + StealthDuck-mix + ambient-cue build (28
intended audio calls, all `ok:true`). Late in the task the agent wanted to list
SoundWaves to consider assigning to the looping cue. The call log shows the
misuse-then-correct exactly:

- `asset.search` `args: "SoundWave search (wrong, needs query)"` →
  `ok:false`, `error_text:"[MISSING_REQUIRED_PARAM] Missing required parameter
  'query' (type: string)"`.
- immediately followed by `asset.search_assets` `args: "SoundWave list"` →
  `ok:true`.

Friction note (verbatim):

> "One wrong call (asset.search needs 'query', used asset.search_assets
> instead)."

The agent had to *know* to swap verbs (it did, in one step) — but a class-only
list ("all SoundWaves") has no natural name `query`, so `asset.search` was the
wrong tool by contract, not by a typo. A single overlay line would have routed
the agent to `search_assets` on the first try. This was the only `is_error`
call in an otherwise clean 42-call task, so it is small-but-real PROCESS
friction, not an outcome bug.

## What it should do / how to fix (docs-first, NAMES the overlay page)

Improve `docs/wiki-src/asset.md`. The overlay edit is a downstream wiki process,
not this audit's job — naming the page and the change is the deliverable:

- Add a short "asset search verbs" note (alongside the existing `asset.list` /
  `asset.dump` entries) that reconciles **all three** overlapping list/search
  verbs and states the split plainly. Crucially, the overlay **already**
  documents a third path — **`asset.list`** with a `filter.class` table
  (`asset.md` "asset.list" section) — which answers the common "list every
  asset of class X" intent with **zero round-trip**, so the note must route that
  intent to a canonical verb rather than mentioning only the two `search*`
  siblings:
  - **`asset.list { filter: { class: "SoundWave" } }`** (or `asset.search_assets
    { classNames:["SoundWave"] }`) — the canonical **class-only list, no name
    pattern**; both are all-optional and need no `query`.
  - **`asset.search`** — use only when you have a **name pattern** (`query`
    **required**, supports `*`/`?` wildcards plus the class/`parentClassPath`
    filters). It errors `MISSING_REQUIRED_PARAM 'query'` if you reach for it
    without a name, so the failure is self-correcting once documented.
  - **`asset.search_assets`** — class/path-driven filter list (all params
    optional, `classNames[]` + `packagePaths[]`, `recursiveClasses` for
    subclasses).
- Explicitly note that **`asset.find` is NOT a verb** — the namespace ships only
  the tag-only `asset.find_by_tag` / `asset.find_objects_by_tag`, so a bare
  `asset.find` errors `[UNKNOWN_ACTION]`; route the caller to the class-list
  verbs above instead.

A cheaper structural option (out of scope for this docs-tagged ticket) would be
to let `asset.search` treat an omitted `query` as a wildcard `*` match (so the
class-only list works through either verb) and/or to converge the two verbs
behind one name with an alias — but the doc note unblocks callers now without
any handler change.

**Workaround:** for a class-only / name-less asset list use the already-documented
`asset.list { filter: { class: "..." } }` (or `asset.search_assets { classNames:[...] }`);
reserve `asset.search` for when you have a name pattern for `query`; `asset.find`
is not a verb.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of an `audio.authoring`
  SoundClass-hierarchy + StealthDuck-mix + ambient-cue build (outcome tool_bug
  for the separate looping no-op judge-filed `B-create-sound-cue-looping-noop`;
  this is the distinct PROCESS angle). The `asset` namespace ships two
  near-synonymous search verbs with divergent required params —
  `asset.search` (`AssetManageHandler.cpp:945`, name-first, **requires
  `query`**) and `asset.search_assets` (`AssetQueryHandler.cpp:166`,
  class/path-first, **all optional**) — and the `asset.md` overlay documents
  **neither by name**, so an agent listing assets of a class reaches for the
  more-general-sounding `asset.search` and hits
  `MISSING_REQUIRED_PARAM 'query'` before correcting to `asset.search_assets`.
  Call log shows the misuse-then-correct in adjacent calls (the only
  `is_error` in an otherwise clean 42-call task). Friction note (verbatim):
  "One wrong call (asset.search needs 'query', used asset.search_assets
  instead)." Same namespace-internal contract-disagreement shape as
  `E-asset-path-vs-assetpath-list-drift` / `E-class-name-format-inconsistency`.
  Proposed: add an "asset search verbs" note to `docs/wiki-src/asset.md` naming
  both verbs and the name-pattern-vs-class-list split, with the
  `asset.search_assets { classNames:[...] }` recipe for the class-only list;
  optionally make `asset.search` treat an omitted `query` as `*`.
- `#2-bare-find-guess-unknown-action` `OPEN` reporter — Independent repro in an
  `audio` runtime-soundscape task (story: place AAmbientSound + UAudioComponent,
  one-shot, fade in/out, push/pop a SoundMix; outcome tool_bug, judge filed the
  separate `B-create-ambient-sound-location-object-dropped`). Same root friction
  (caller wants a class-only asset list, doesn't know the verb, pays a
  trial-and-error round-trip), with a **stronger variant**: the agent's first
  guess was the bare **`asset.find`**, which does not exist at all and errored
  `[UNKNOWN_ACTION] Unknown action: asset.find` **twice** (once filtering for
  `SoundMix`, once for `SoundClass`) before it reached `asset.search_assets`.
  This is worse than the #1 `asset.search`-needs-`query` round-trip because
  `asset.find` is an especially natural guess — the namespace already ships
  `asset.find_by_tag` and `asset.find_objects_by_tag`, so a `find` verb *looks*
  like it should exist, and there is no top-level `asset.find` to redirect.
  Friction note (verbatim): *"there's no asset.find verb (the obvious guess) so
  it errored UNKNOWN_ACTION twice before I found asset.search_assets."* Same fix
  unblocks both: the `docs/wiki-src/asset.md` "asset search verbs" note should
  name `asset.search_assets { classNames:[...] }` as the canonical class-list
  recipe AND explicitly note that the intuitive `asset.find` is **not** a verb
  (the `find_*` siblings are tag-only), routing the caller to `search_assets` on
  the first try. Aggregated here rather than re-filed.
- `#3-docs-reword-and-fix` `IN-REVIEW` developer — Adopted the adversarial-lens
  scope correction: the overlay **already** documents `asset.list` with a
  `filter.class` table, which answers the "list every asset of class X" intent
  with zero round-trip, so a note naming only the two `search*` siblings would
  have been a band-aid. Reworded the ticket title/**Fix:**/Workaround to
  reconcile **all three** list/search verbs (`asset.list` / `asset.search` /
  `asset.search_assets`) and the phantom `asset.find`. Implemented in
  `Docs/wiki-src/asset.md`: (a) a new "List / search verbs" bullet under
  `## Cross-cluster overlap` (renders on the asset namespace page) that routes a
  class-only list to `asset.list { filter:{class} }` or `asset.search_assets
  { classNames:[...] }`, reserves `asset.search` for a name `query` (noting it
  errors `[MISSING_REQUIRED_PARAM] 'query'` without one), and states `asset.find`
  is not a verb (only the tag-only `find_by_tag` / `find_objects_by_tag` exist);
  (b) new `### asset.search` and `### asset.search_assets` H3 method-page sections
  with the per-verb param contracts and the class-list recipe. Regression test:
  `FWikiHandlerAssetSearchVerbsReconciledTest`
  (`…/Private/Tests/Infra/TestWikiHandler.cpp`,
  `EditorAutomationRpcGateway.infra.wiki_handler.MethodPage.AssetSearchVerbsReconciled`)
  drives production `WikiHandler::RenderPage` for `asset` / `asset.search` /
  `asset.search_assets` and asserts the overlay-exclusive markers (`is not a
  verb`, `asset.list { filter: { class:`, `[MISSING_REQUIRED_PARAM]`, `no-name
  path`, `classNames:["SoundWave"]`) — fails if any overlay edit is reverted.
  Docs-only; no handler change.
