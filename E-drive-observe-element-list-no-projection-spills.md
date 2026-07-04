---
id: E-drive-observe-element-list-no-projection-spills
title: "drive.observe element list has no compact/byte-aware projection — full Slate/UMG handle paths overflow the inline budget even with interactables_only + max_elements"
status: OPEN
severity: Medium
category: ergonomic
tags: [drive, observe, editor_chrome, response-size, response-spill, projection, oversized, docs]
encounters: 1
lastSeen: 2026-07-04T19:39:32.1148071+03:00
---

# `drive.observe` element list has no compact/byte-aware projection — the verbose per-element handle path spills even a capped observe

`drive.observe` is the primary "what controls are on this surface?" inspection
verb. On `surface:editor_chrome` (and equally on `surface:game` / `surface:web`)
its per-element `handle` is the **entire widget-tree path** — for editor_chrome
the full Slate path (e.g.
`Area[0]/SOverlay[0]/SHorizontalBox[0]/SOverlay[1]/SSplitter[0]/SDockingTabStack[0]/SVerticalBox[0]/.../SButton[2]`),
for game/web the nested UMG path
(e.g. `W_OverallUILayout_C_0/W_MyDrones_C_0/W_DroneSelectionButton_C_8/CustomiseButton/SCommonButton`).
These long handle strings dominate the payload, so the observation blows the
10000-char inline display budget and spills to
`Saved/PinWright/HttpResponses/.../<uuid>.json` (`E-http-response-spill`, DONE),
forcing an extra `Read` for even a trivial "is the toolbar present?" presence
check.

The natural narrowing levers **do not** bring it under threshold:
`interactables_only:true` and `max_elements:N` bound the element **count**, not
the **byte size** — a handful of capped elements, each carrying its full
widget-tree path, already exceeds 10000 chars. So unlike the list verbs where a
smaller `limit`/tighter filter keeps a page inline, here the caller's only
recovery after applying both caps is still a spill-file Read (or worse; see the
game/web evidence below).

## What it should do

Give `drive.observe` a way to keep a presence/scan observe inline:
- a **compact / labels-only projection** (return element `label` + `type` +
  short synthetic id, omitting the full Slate/UMG handle path unless asked), or
- a **shorter opaque handle** (a stable index/token that resolves server-side)
  instead of the verbose tree path, or
- **byte-aware truncation** (cap the serialized payload by bytes, not just
  element count, and report `omitted_count` — which `drive.observe` already does
  correctly), or
- a **subtree-rooted / name-filtered observe** so "does control X exist under
  window Y" narrows server-side rather than returning the whole frame.

Any one keeps the common "confirm a control is there" check inline; the
full-handle rows stay available opt-in for the cases that actually act on a
handle. Mirrors the per-method narrowing levers already landed/proposed across
the `*-no-projection-spills` family (`E-actor-list-no-limit-spills` landed;
`E-blueprint-list-no-projection-spills`, `E-inspect-object-no-projection-spills`,
`E-get-node-details-batch-no-projection-spills` proposed).

**Docs page:** `docs/wiki-src/drive.md` (the `drive.observe` section) — once a
projection/compact lever lands, document that it trims the per-element handle so a
presence/scan observe stays inline; until then, note that `max_elements` caps count
not bytes and that a capped editor_chrome observe still commonly spills.

## Distinct from

- `F-drive-observe-screenshot-inline-base64` (IN-REVIEW) — **same method, different
  fix family.** That ticket is strictly the Set-of-Mark **screenshot** file-delivery
  mode (route the base64 PNG to a path). Its own **Scope** section explicitly splits
  THIS element-list overflow out ("belongs to the response-spill /
  `*-no-projection-spills` family ... it should be filed on its own"), and its
  History `#2`/`#3` recorded the prior element-list evidence but disclaim fixing it.
  This ticket is that separately-filed element-list projection gap. Resolving the
  screenshot delivery does not clear it (the element list dominates the payload even
  with `screenshot:false`).
- `E-http-response-spill` (DONE) — the generic server-side spill-to-disk mechanism
  itself; this ticket is that a specific verbose reader lacks a projection to stay
  under the threshold in the first place.

## Evidence

From the `drive.list_windows` asset-editor smoke-check struggle audit (seed
`drive.list_windows`, namespace `drive`, outcome **ergo** — the seed itself was
clean: open/observe/close of `BP_Button_Parent` round-tripped and `list_windows`
correctly showed the 3->4->3 window transition; the friction was the incidental
observe step). The doer's already-capped
`drive.observe {surface:editor_chrome, window_title:BP_Button_Parent,
screenshot:false, interactables_only:true, max_elements:40}` — both narrowing caps
applied and `screenshot:false` — **still** returned **61238 chars** (>10000
threshold, ~6.1x over) and spilled to `Saved/PinWright/HttpResponses/.../<uuid>.json`
(`omitted_count` 236, correctly reported), forcing a separate `Read` of the spill
file just to confirm the Blueprint editor toolbar (`Compile`/`Save`/`Find`/`Class
Defaults`) was present. Agent friction note, verbatim: *"drive.observe responses
exceeded the 10000-char inline budget and spilled to Saved/PinWright/HttpResponses
JSON files, forcing extra Reads ... not a blocker, and observe correctly reports
omitted_count (236)."* CallAnalyzer inefficiency #2 (pattern=workaround), verbatim:
*"The cap bounds element COUNT but not byte size: each editor_chrome element's
handle is the full Slate widget-tree path ... so ~40 capped elements alone blow the
budget."*

Prior cross-surface occurrences of the same element-list overflow (recorded in
`F-drive-observe-screenshot-inline-base64` History `#2`/`#3`, before its scope-split):
`surface:editor_chrome` full-frame observes at ~922KB-1.75MB each (5 observes, all
spilled); `surface:game` at 23898/24928 chars for only ~26 elements;
`surface:web` at 15344/26549 chars — in every case `interactables_only:true` +
`max_elements` did NOT bring the payload under threshold (long nested handle paths
dominate), and one game/web recovery needed an out-of-band Python script that hit a
`cp1251` `UnicodeDecodeError` on Cyrillic element labels and required an explicit
utf-8 open. This task is a cleaner single-Read datapoint on the same root cause.

severity rationale: impact=soft-blocker — the two natural narrowing levers
(`interactables_only`, `max_elements`) cap element count but not payload bytes, so a
capped presence-check observe still spills and recovery ranges from one spill-file
`Read` (this task) to an out-of-band script + utf-8 fix (game/web evidence) ×
reach=`drive.observe` on editor_chrome/game/web is an every-session drive-navigation
path (reach bump) -> Medium. (Consistent with the Medium already assigned to this
same sub-issue in `F-drive-observe-screenshot-inline-base64` #3.)

## History
- `#1-initial-audit` `OPEN` reporter — Filed as the separate element-list ticket that `F-drive-observe-screenshot-inline-base64`'s Scope section calls for. From the `drive.list_windows` asset-editor smoke-check struggle audit (seed `drive.list_windows`, namespace `drive`, outcome **ergo** — the seed method itself was zero-friction: `list_windows` correctly detected the new `BP_Button_Parent` window post-open and its removal post-close, 3->4->3). CallAnalyzer inefficiency #2 (pattern=workaround, type E) on `drive.observe`: an already-capped `drive.observe {surface:editor_chrome, window_title:BP_Button_Parent, screenshot:false, interactables_only:true, max_elements:40}` returned **61238 chars** (>10000 threshold) and spilled to `Saved/PinWright/HttpResponses/.../<uuid>.json` (`omitted_count` 236), forcing an extra `Read` for a simple toolbar-presence check. Root cause: the per-element `handle` is the full Slate widget-tree path, so `max_elements` caps count but not bytes and a capped observe still overflows. Proposed: a compact/labels-only projection, a shorter opaque handle, byte-aware truncation, or a subtree/name-filtered observe so a presence/scan check stays inline; document in `docs/wiki-src/drive.md`. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — no existing drive.observe element-list ticket; `F-drive-observe-screenshot-inline-base64` (IN-REVIEW) is the same method but the screenshot-delivery fix family and explicitly splits this out; `E-http-response-spill` (DONE) is the generic spill mechanism. Prior cross-surface element-list evidence (editor_chrome ~922KB-1.75MB; game 23898/24928 for ~26 elements; web 15344/26549; caps ineffective; one recovery needed an out-of-band utf-8 script) lives in `F-drive-observe-screenshot-inline-base64` #2/#3. Seeded `encounters: 1` (new file); `lastSeen` set.
