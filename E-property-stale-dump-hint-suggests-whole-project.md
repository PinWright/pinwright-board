---
id: E-property-stale-dump-hint-suggests-whole-project
title: "property.get / property.list stale-dump hint leads with asset.dump_folder {folderPath:'/Game'} — a whole-project dump as the default remedy for one stale asset, on an editor several agents share"
status: OPEN
severity: Low
category: ergonomic
tags: [property, property-get, property-list, hint, asset-dump, dump-folder, stale-dump, shared-editor, ergonomics, weapons]
encounters: 1
lastSeen: 2026-09-05T00:00:00Z
---

# The remedy offered for one stale asset is a dump of everything

When `property.get` / `property.list` detect that the asset's dump is stale, the response's `hint`
recommends:

```
asset.dump_folder {"folderPath": "/Game"}
```

That is the whole project. The caller's problem is one asset, and the narrowed single-asset form
(`asset.dump {assetPath: …}`) does the job — but it is not what the hint leads with, and a hint is
read as the recommended next call.

## Why the wide form is the wrong default

- **Cost is unbounded and unrelated to the request.** A `/Game` sweep is minutes-to-hours of work to
  refresh one asset's sidecars; the board already carries a family of tickets about `dump_folder`
  scale (`B-dump-folder-unbounded-asset`, `B-asset-dump-folder-no-completion-signal`,
  `B-asset-dump-game-high-skip-ratio`, `B-dumpcache-staleness-loop`).
- **This editor is shared.** Several agents work one editor instance concurrently. A `/Game` dump
  taken to satisfy one `property.get` occupies the game thread for everyone, and nothing in the hint
  suggests the caller is about to do that on someone else's behalf as well as their own.
- **The narrow form is strictly better here.** The staleness the hint reacts to is per-asset, and
  the caller already has the asset path in hand — it is in the request that produced the hint.

## What is asked for

1. **Lead with the narrowed form.** Emit `asset.dump {assetPath: "<the path from this request>"}` —
   interpolated, not a template, since the handler has the path. Mention `dump_folder` afterwards, if
   at all, as the option for refreshing a whole tree.
2. **Do not name `/Game` as a literal.** If a folder form is kept in the text, use the asset's own
   parent folder rather than the project root; a copy-pasteable `/Game` is the specific thing that
   makes this expensive.
3. **One line on cost.** If the wide form stays in the hint, it should say it dumps every asset under
   the path.

## Severity

**Low** — pure friction and self-correcting for a caller who thinks about it. Held at Low rather than
lower because the hint is on `property.get` / `property.list`, which run in nearly every session, and
because the cost is paid by other agents on the same editor rather than by the caller who follows the
advice.

## Related

- `B-dump-folder-unbounded-asset`, `B-asset-dump-folder-no-completion-signal`,
  `B-asset-dump-game-high-skip-ratio`, `B-dumpcache-staleness-loop`,
  `F-asset-dump-folder-skip-levels-by-default` — the existing evidence that a `/Game` sweep is
  expensive and awkward to run; this ticket is about recommending it as a reflex.
- `E-asset-dump-folder-cache-miss-reason-opaque` — the adjacent question of *why* a dump reads stale,
  which is what a caller would want before choosing a remedy at all.
- `E-rendering-unknown-property-no-hint` — the opposite complaint on a neighbouring surface (no hint
  where one is wanted), i.e. the hint machinery is worth getting right rather than removing.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Observed during a WEAPONS critic review round 3. `property.get` / `property.list` emit a stale-dump `hint` whose recommended remedy is `asset.dump_folder {"folderPath":"/Game"}` — a whole-project dump — for a staleness condition that is per-asset and whose asset path is already in the request that produced the hint. On this project the editor is shared by several agents concurrently, so following the hint occupies the game thread for everyone, and the board already carries a family of tickets on `dump_folder` scale and completion signalling (`B-dump-folder-unbounded-asset`, `B-asset-dump-folder-no-completion-signal`, `B-asset-dump-game-high-skip-ratio`, `B-dumpcache-staleness-loop`). Ask: lead with the narrowed `asset.dump {assetPath:"<path from this request>"}` interpolated rather than templated, since the handler has the path; if a folder form is kept, use the asset's parent folder rather than a copy-pasteable `/Game` literal; and if the wide form stays, add one line saying it dumps every asset under the path. Severity **Low** — pure friction, self-correcting for a caller who stops to think — held at Low rather than lower because these two verbs run in nearly every session and because the cost of following the advice is paid by the other agents sharing the editor. No plugin source was opened for this ticket and no `file:line` is claimed; the hint text is quoted from the live responses.
