---
id: E-drive-observe-no-label-filter
title: "drive.observe has no text/label filter — finding a control by its label means grepping the spilled element JSON by hand"
status: OPEN
severity: Low
category: ergonomic
tags: [drive, observe, filter, label, discovery, response-spill]
encounters: 1
lastSeen: 2026-07-17T13:15:59+03:00
---

# `drive.observe` has no text/label filter — locating one control by its label forces a hand-grep of the spilled element list

`drive.observe` enumerates *every* addressable element on a surface. Its only
narrowing levers today are count/size caps — `interactables_only`,
`max_elements`, and `max_bytes` (the sibling `E-drive-observe-element-list-no-projection-spills`
byte-cap). None of them selects elements **by their text**. A caller whose intent
is "find the button labelled X and get its handle to click it" has no
`filter` / `label_contains` / `query` param to reach for; the natural guess is
rejected:

```
drive.observe {filter:"СОЗДАТЬ"} → [UNKNOWN_PARAMS] Unknown parameter(s) for 'drive.observe': [filter]
```

So the caller must observe the *whole* surface, let the verbose element list spill
to `Saved/PinWright/HttpResponses/.../<uuid>.json` (the handle is the full
widget-tree path, so a rich menu overflows the inline budget — see the sibling
ticket), and then grep the spill file by hand to find the one element whose label
matches. The spill JSON is a single line with escaped quotes, so the grep loop is
awkward: in one session it took **~6 extra Bash calls**, including `dd`
byte-offset extraction, to pull button handles out by their Russian labels.

This is a distinct lever from the byte-cap in
`E-drive-observe-element-list-no-projection-spills`. That ticket bounds *how many*
bytes of the element list come back (dropping the remainder into `omitted_count`);
it does not let you say *which* elements you want. Worse, the two interact badly:
under a byte cap the target element may land in the dropped `omitted_count`
remainder, so even the shipped byte-cap doesn't guarantee your labelled control is
in the returned slice. A server-side label filter returns the matching element(s)
regardless of position, and — because it shrinks the list to a handful — usually
keeps the response inline without needing `max_bytes` at all.

## What it should do

Add a substring filter param to `drive.observe` (e.g. `label_contains` / `query`,
with the `filter` alias the caller already guessed) that matches server-side,
case-insensitively, against each element's `Label` (and, ideally, `Handle`), and
returns only matching elements. It applies before the screenshot marks and before
the count/byte caps, mirroring how `actor.list`'s `filter` narrows a list verb by
label substring (`E-actor-list-no-class-filter`). This is the drive-surface analog
of the same name-substring narrowing that already exists on the actor/asset list
verbs. Document it in `docs/wiki-src/drive.md` alongside the `max_bytes` guidance.

## Distinct from

- `E-drive-observe-element-list-no-projection-spills` (IN-REVIEW) — **same method,
  different lever.** That ticket added `max_bytes`, a *byte-size* cap on the element
  list; this ticket asks for a *text/label* filter that selects *which* elements
  come back. Its History `#3` records the session evidence for both and explains why
  the label filter is filed separately rather than folded into the IN-REVIEW byte-cap.
- `E-actor-list-no-class-filter` (IN-REVIEW) — the same missing-narrowing-param
  ergonomic shape on a different verb; `actor.list` at least *has* a name-substring
  `filter`, which is exactly the lever `drive.observe` lacks.

## Evidence

From a 2026-07-17 UI-drive session (a ~183-element menu with Russian button
labels). The caller guessed `drive.observe {filter:"СОЗДАТЬ"}` and hit
`[UNKNOWN_PARAMS] Unknown parameter(s) for 'drive.observe': [filter]`. With no
server-side filter, the recovery was to observe the whole surface, let the element
list spill, then grep the one-line quote-escaped spill JSON by hand — ~6 extra Bash
calls including a `dd` byte-offset extraction — to recover the button handles by
their labels. A `label_contains`/`query`/`filter` param would have returned the one
matching element (and its actionable handle) in a single inline call.

severity rationale: impact ≈ **soft blocker** (doable, but only via many extra
calls / a hand-grep of a spill file — README rubric Medium band), reduced one level
for **reach** (`drive.observe` is one drive-namespace verb among ~a dozen, and the
label-lookup case is a subset of its uses; the shipped `max_bytes` + inline
eyeballing already covers many discovery needs). Net -> **Low**, consistent with the
sibling `E-actor-list-no-class-filter` (also a missing-filter-param ergonomic, Low)
and the whole `drive.observe` `*-no-projection-spills` family. Flagged for retriage
to **Medium** if the ~6-call hand-grep cost recurs across sessions.

## History
- `#1-initial-report` `OPEN` reporter — Filed from a 2026-07-17 UI-drive session (~183-element menu, Russian labels). `drive.observe` has no text/label filter: `drive.observe {filter:"СОЗДАТЬ"}` → `[UNKNOWN_PARAMS] ... [filter]` (confirmed absent — no `filter`/`label_contains`/`query`/`text_filter`/`name_filter` param across the Drive handlers). To find one labelled control the caller had to observe the whole surface, let the verbose element list spill to `Saved/PinWright/HttpResponses/...json`, and hand-grep the one-line quote-escaped JSON (~6 extra Bash calls, incl. `dd` byte-offset extraction) to recover button handles by their Russian labels. Distinct lever from `E-drive-observe-element-list-no-projection-spills` (IN-REVIEW), which added `max_bytes` — a *byte-size* cap, not a *which-elements* text filter; under that byte cap the target can even land in the dropped `omitted_count` remainder, so a label filter is still needed. Proposed: add a server-side case-insensitive substring filter (`label_contains`/`query`, `filter` alias) over `Label` (and `Handle`), applied before the screenshot marks and the count/byte caps — the drive analog of `actor.list`'s `filter`; document in `docs/wiki-src/drive.md`. Split out of `E-drive-observe-element-list-no-projection-spills #3` rather than folded into that IN-REVIEW byte-cap fix. Dedup: grep across the board found no existing drive.observe text/label-filter ticket. Seeded `encounters: 1`; `lastSeen` set.
