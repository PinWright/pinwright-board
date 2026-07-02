---
id: E-sequencer-list-no-folder-filter-spills
title: "sequencer.list has no folder/path filter or limit — it always scans the whole /Game tree, so a folder-scoped 'list sequences in /Game/Cinematics' intent dumps the project-wide list and spills to file"
status: OPEN
severity: Low
category: ergonomic
tags: [sequencer, response-size, oversized, pagination, projection, folder-filter, docs]
encounters: 2
lastSeen: 2026-07-02T09:55:31.2017475+03:00
---

# sequencer.list has no folder/path filter or limit — project-wide dump spills to file

`sequencer.list` is the obvious "what level sequences exist?" entry point, and a
very common reason to call it is to confirm sequences **in one folder** (e.g.
verify a rename took inside `/Game/Cinematics`). But the method takes
**`RPC_NO_PARAMS`** (`SequenceHandler.cpp:1364-1365`) and hardcodes a project-wide
scan: `Filter.PackagePaths.Add(FName("/Game"))` with `bRecursivePaths = true`
(`SequenceHandler.cpp:1377-1378`), enumerating **every** `ULevelSequence` under
`/Game` (`AssetRegistry.GetAssets`, :1380-1381). There is **no `path`/folder
filter, no `limit`, no pagination cursor, and no name filter** — the caller cannot
ask "just the sequences in `/Game/Cinematics`" or "just the first N".

On a populated project the full sequence list crosses the **10000-char inline
spill threshold**, so the response comes back as `outputTooLong` and the full
payload is written to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`,
forcing the caller to **Read/Grep the spilled file** just to confirm a single
folder's sequences. The story that surfaced this asked literally for *"list the
level sequences in /Game/Cinematics … to confirm the old name is gone and the new
TrailerSeq_v01 exists"* — a one-folder, two-name check — yet the only available
call was the project-wide `sequencer.list {}`, which spilled and had to be grepped.

## What's wrong

`SequenceHandler.cpp:1364` registers `RPC_NO_PARAMS`. The body builds a fixed
`FARFilter` rooted at `/Game` with `bRecursivePaths=true` (:1374-1378) and emits
one `{path, name}` per sequence (:1385-1392) with no cap. Unlike the sibling
verbose readers, the **per-row payload is already minimal** (`path` + `name`
only), so a `fields`/`namesOnly` projection would not help much here — the bytes
come from sheer **count**, which the missing folder scope directly compounds: a
folder-scoped intent is forced to pull the entire project's sequences.

## What it should do

Mirror the fix family already shipped/proposed for the sibling verbose readers
(`E-recorder-list-sessions-limit`, `E-graph-connections-pagination`,
`E-actor-list-no-limit-spills`, `E-skeleton-list-bones-no-limit-spills`,
`E-volume-get-info-no-limit-spills`, `E-gameplay-tags-list-default-limit-spills`),
plus the folder-scope angle specific to this method:

- Add an optional **`path`** (package path; default `/Game`) so a folder-scoped
  listing — `sequencer.list {path:"/Game/Cinematics"}` — scans only that subtree
  and stays inline. This is the primary ask for this RPC and the one the family
  siblings don't all need (most are level-/asset-scoped already). (Note
  `asset.list` already supports a `path`/`filter` for this; `sequencer.list` is
  the convenience reader that should grow the same scope param — heed
  `E-asset-list-path-ignored`'s lesson that the path must actually be honored.)
- Add an optional **`limit`** (default small enough to stay inline; `0` = all)
  that truncates after iteration while a `count`/`totalCount` keeps reporting the
  untruncated total so elision is detectable — exactly the contract
  `E-recorder-list-sessions-limit` (#2/#4) and `E-graph-connections-pagination`
  (#2/#3) landed.
- Optionally a name **`filter`** substring (since rows are already `{path,name}`,
  a name filter is more useful than a `fields` projection here).
- **Docs (`docs/wiki-src/sequencer.md`):** add a `### sequencer.list` note that
  the method scans the entire `/Game` tree, that the result can exceed the inline
  budget and spill to file on a populated project, and that the proposed
  `path`/`limit` are the way to keep a folder-scoped listing inline.

## Evidence

From the trailer-cinematic naming task (focus `sequencer.rename`, namespace
`sequencer`, outcome `clean`, `filed_id` empty — the judge filed nothing and the
self-reported friction was *"none"*). This response-size + missing-folder-scope
friction is a distinct PROCESS angle the friction note flagged only in passing:
verbatim — *"only minor handling was that sequencer.list output exceeded the
display limit and was dumped to a file I grepped (it is project-wide with no
folder filter param)"*. Call-log: the `sequencer.list` call (#7 of 9,
`args_summary:"{} (project-wide; dumped to file)"`) overflowed the inline budget
and spilled to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, then was
grepped — when the task's intent was the tiny "confirm the two names under
`/Game/Cinematics`". Source confirms the cause: `RPC_NO_PARAMS` + hardcoded
`/Game` recursive scan (`SequenceHandler.cpp:1364-1378`). (The friction note's
other remark — deliberately choosing `sequencer.rename` over `asset.rename` to
avoid a redirector — was a *successful avoidance*, not friction: no retry, no
error, no extra step; not filed.)

## Distinct from

- `E-http-response-spill` (DONE) — the *generic* server-side spill mechanism (the
  file-reference fallback itself); this ticket is that a *specific verbose reader*
  has no narrowing to stay under the threshold in the first place, the same
  relationship the rest of the no-limit-spills family has to the spill mechanism.
- `E-actor-list-no-limit-spills` / `E-volume-get-info-no-limit-spills` /
  `E-skeleton-list-bones-no-limit-spills` / `E-gameplay-tags-list-default-limit-spills`
  (OPEN) — same *shape* (verbose reader → spill → forced Read) and same
  `limit`/`count` fix, but on different methods/handlers. This one's standout is
  the **missing folder/`path` scope** (the method is hardcoded to the whole
  `/Game` tree) plus an already-minimal per-row payload, so the fix leans on
  `path`+`limit` rather than a `fields` projection.
- `F-rpc-sequencer-list-sections` (DONE) / `E-sequencer-list-tracks-omits-camera-cut-undocumented`
  (OPEN) — different `sequencer.list_*` readers about *track/section* content of
  one sequence; this is the asset-enumeration reader `sequencer.list`, a different
  RPC with a different (count/scope) gap.
- `E-asset-list-path-ignored` (DONE) — the general `asset.list` path-honoring bug;
  cited here because the proposed `path` param on `sequencer.list` should honor
  the path rather than silently default to `/Game` the way that ticket's bug did.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the trailer-cinematic naming struggle audit (focus `sequencer.rename`, namespace `sequencer`, outcome clean; the judge filed nothing and the self-reported friction was "none"). Distinct PROCESS angle from the friction note: `sequencer.list` has no folder/`path` filter, no `limit`, no name filter — it is `RPC_NO_PARAMS` (`SequenceHandler.cpp:1364-1365`) and hardcodes a recursive `/Game`-wide `ULevelSequence` scan (`SequenceHandler.cpp:1374-1381`, emitting `{path,name}` per row :1385-1392 with no cap). The task's intent was the folder-scoped "list the level sequences in /Game/Cinematics to confirm the rename", but the only call available was project-wide `sequencer.list {}` (call #7 of 9, `args_summary:"{} (project-wide; dumped to file)"`), which crossed the 10000-char inline threshold, spilled to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, and was grepped. Friction note verbatim: "sequencer.list output exceeded the display limit and was dumped to a file I grepped (it is project-wide with no folder filter param)". Proposed: add a `path` (folder-scope, default /Game), a `limit` (default-inline, 0=all, with untruncated `count`) per `E-recorder-list-sessions-limit`/`E-graph-connections-pagination`, optionally a name `filter`, and document the whole-/Game-scan/overflow behavior in the `### sequencer.list` section of `docs/wiki-src/sequencer.md`. Dedup: ripgrep across OPEN/DONE/WONTFIX — no ticket references `sequencer.list` (the asset-enumeration reader); `F-rpc-sequencer-list-sections`/`E-sequencer-list-tracks-omits-camera-cut-undocumented` are the per-sequence track/section readers; `E-actor-list-no-limit-spills`/`E-volume-get-info-no-limit-spills`/`E-skeleton-list-bones-no-limit-spills`/`E-gameplay-tags-list-default-limit-spills` are the same shape on other RPCs (this one adds the missing-folder-scope root cause and has an already-minimal per-row payload); `E-http-response-spill` (DONE) is the spill mechanism; `E-asset-list-path-ignored` (DONE) is the general `asset.list` path-honoring lesson. The redirector-avoidance remark in the friction note was a successful avoidance, not friction — not filed.
- `#2-liveness` `OPEN` reporter — still reproduces at HEAD (struggle-audit of the SEED-mode `sequencer.delete` cinematic task, focus `sequencer.delete`, `/Game/Cinematics/IntroMaster`, 23 calls). Notably `sequencer.list` was called **twice in one task** — a folder-scoped inventory before deletion and a re-list after — and BOTH overflowed the inline budget (`22518` and `22062` chars), each spilling to `Saved/EditorAutomation/HttpResponses/*.json` and each forcing a follow-up Grep (`/Game/Cinematics/[A-Za-z_]+`) to isolate just the three Cinematics sequences the task cared about. Same missing-folder-scope + no-limit spill root cause; the twice-per-task hit doubled the Read/Grep cost. No new angle — the proposed `path`/`limit`/name-`filter` fix (with untruncated `count`) still stands. Friction note verbatim: "sequencer.list twice exceeded the 10k display limit and spilled to a file, needing a grep to check the Cinematics folder."
