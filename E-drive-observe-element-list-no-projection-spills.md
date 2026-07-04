---
id: E-drive-observe-element-list-no-projection-spills
title: "drive.observe element list has no byte-size cap — full Slate/UMG handle paths overflow the inline budget on a discovery/scan observe even with interactables_only + max_elements"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [drive, observe, editor_chrome, response-size, response-spill, projection, byte-cap, oversized, docs]
encounters: 1
lastSeen: 2026-07-04T19:39:32.1148071+03:00
---

# `drive.observe` element list has no byte-size cap — the verbose per-element handle path spills even a count-capped observe

`drive.observe` is the drive surface's "what controls are on this surface?" inspection
verb. On `surface:editor_chrome` (and equally on `surface:game` / `surface:web`) its
per-element `handle` is the **entire widget-tree path** — for editor_chrome the full
Slate path (e.g.
`Area[0]/SOverlay[0]/SHorizontalBox[0]/SOverlay[1]/SSplitter[0]/SDockingTabStack[0]/SVerticalBox[0]/.../SButton[2]`),
for game/web the nested UMG path
(e.g. `W_OverallUILayout_C_0/W_MyDrones_C_0/W_DroneSelectionButton_C_8/CustomiseButton/SCommonButton`).
These long handle strings dominate the payload, so the observation blows the
10000-char inline display budget and spills to
`Saved/PinWright/HttpResponses/.../<uuid>.json` (`E-http-response-spill`, DONE),
forcing an extra `Read`.

The two count-narrowing levers **do not** bring it under threshold:
`interactables_only:true` and `max_elements:N` bound the element **count**, not the
**byte size** — a handful of capped elements, each carrying its full widget-tree path,
already exceeds 10000 chars (`FilterObservationElements` only dropped-by-interactable
then `SetNum(MaxElements)` — no byte bound; `WriteElement` always emits the full
`handle`). So unlike the list verbs where a smaller `limit`/tighter filter keeps a page
inline, before this fix the caller's only recovery after applying both caps was still a
spill-file `Read`.

## Scope — this is the discovery/scan case, not a presence check

A **pure presence check** ("is control X on this surface?") does **not** need
`drive.observe` at all: `drive.expect` with a `{type:widget_present, target:"Compile"}`
condition answers it synchronously and inline, returning the small fixed shape
`{ met, actual, expected, detail, root_name }` that **cannot** spill (no element list),
matching the target by `label` (case-insensitive) so no handle path is needed
(`DriveExpectHandler.cpp:75-81`, `DriveConditionEval.h:11-14`). `drive.wait_for
widget_present` is its polling twin. The original repro below was really a presence check
(confirm the BP toolbar was present) that `drive.expect` covers — so that framing is
withdrawn.

What `drive.expect`/`wait_for` do **not** serve is **discovery / survey**: enumerate the
(unknown) controls on a surface to pick one and obtain its `handle` to act on. That
genuinely needs the element list, and that is the case this ticket keeps: a survey observe
of a rich editor window still overflows.

## Fix (implemented)

Added a **byte-aware truncation** lever, `max_bytes` (default `0` = unlimited, fully
backward-compatible), to `drive.observe` — the element-list analog of the list verbs'
`limit`+`truncated`, reusing the existing `omitted_count` machinery. It caps the elements'
**total serialized size**, not just their count, so a survey observe stays inline instead
of spilling; the dropped remainder is reported in `omitted_count` (never a silent
truncation), and at least one element is always returned.

Byte-aware truncation is the right lever here (rather than the sibling family's
`namesOnly`/`fields` per-row projection) because `drive.observe`'s verbose field and its
**actionable-identity** field are the **same** field: for editor_chrome
`Element.Handle == Element.Path == BuildWidgetPath(...)` (`DriveEditorChrome.cpp:172`), and
every action verb resolves a target **by handle** (`DriveActionCommon.cpp:188-192`). So —
unlike `actor.list`, where `namesOnly` drops the verbose `path` but keeps the actionable
`name` — you cannot drop the handle without breaking discovery-to-act. Capping bytes (and
reporting the remainder) preserves the actionable handles that survive while bounding the
payload regardless of how deep any single path is. (A future, larger option is a
**shorter opaque server-resolvable handle**, which would shrink every row and let more fit
inline; that touches the resolver and every action verb, so it is out of scope for this
fix.)

**Docs:** `docs/wiki-src/drive.md` (the `drive.observe` section) documents `max_bytes` and
cross-references `drive.expect widget_present` for the presence-check case; the `max_bytes`
RPC param description carries the same guidance inline.

## Distinct from

- `F-drive-observe-screenshot-inline-base64` (IN-REVIEW) — **same method, different
  fix family.** That ticket is strictly the Set-of-Mark **screenshot** file-delivery
  mode (route the base64 PNG to a path). Its own **Scope** section explicitly splits
  THIS element-list overflow out ("belongs to the response-spill /
  `*-no-projection-spills` family ... it should be filed on its own"), and its History
  `#2`/`#3` recorded the prior element-list evidence but disclaim fixing it. Its
  screenshot fix is already in HEAD (`screenshot_mode` at `DriveObserveHandler.cpp:21`)
  and does not touch the element list. This ticket is that separately-filed element-list
  gap; the two seams are orthogonal.
- `E-http-response-spill` (DONE) — the generic server-side spill-to-disk mechanism
  itself; this ticket is that a specific verbose reader lacked a byte cap to stay under
  the threshold in the first place.
- The `*-no-projection-spills` family (`E-actor-list-no-limit-spills` IN-REVIEW,
  `E-blueprint-list-no-projection-spills`, `E-inspect-object-no-projection-spills`,
  `E-asset-list-no-projection-spills`, `E-get-node-details-batch-no-projection-spills`,
  `E-widget-describe-no-projection` — all **Low**) — same rubric band; those use a
  `namesOnly`/`fields` projection because their verbose column is *not* the actionable
  key, so this one uses a byte cap instead (see Fix).

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
file. (That particular read was a toolbar-presence check now better served by
`drive.expect widget_present` — see Scope — but the same overflow hits a genuine survey
observe.) CallAnalyzer inefficiency #2 (pattern=workaround), verbatim: *"The cap bounds
element COUNT but not byte size: each editor_chrome element's handle is the full Slate
widget-tree path ... so ~40 capped elements alone blow the budget."*

Prior cross-surface occurrences of the same element-list overflow (recorded in
`F-drive-observe-screenshot-inline-base64` History `#2`/`#3`, before its scope-split):
`surface:editor_chrome` full-frame observes at ~922KB-1.75MB each (5 observes, all
spilled); `surface:game` at 23898/24928 chars for only ~26 elements;
`surface:web` at 15344/26549 chars — in every case `interactables_only:true` +
`max_elements` did NOT bring the payload under threshold (long nested handle paths
dominate). `max_bytes` applies to all three surfaces.

severity rationale: impact=**friction** — the README rubric's Low band names this exact
case ("a response spill that only forces a `Read`", README.md:203-204). Recovery is a
single spill-file `Read` for the surviving discovery case; the sharper presence-check
recoveries in the cross-surface evidence are now served inline by `drive.expect`. reach:
`drive.observe` is one drive-namespace verb among ~a dozen, not an every-session path — no
reach bump (the higher-reach core-inspection siblings `actor.list`/`asset.list`/
`inspect_object` are all Low). -> **Low**, consistent with the entire
`*-no-projection-spills` family. (Downgraded from the Medium mis-assigned by borrowing the
cross-surface recovery cost and an unjustified reach bump.)

## History
- `#1-initial-audit` `OPEN` reporter — Filed as the separate element-list ticket that `F-drive-observe-screenshot-inline-base64`'s Scope section calls for. From the `drive.list_windows` asset-editor smoke-check struggle audit (seed `drive.list_windows`, namespace `drive`, outcome **ergo** — the seed method itself was zero-friction: `list_windows` correctly detected the new `BP_Button_Parent` window post-open and its removal post-close, 3->4->3). CallAnalyzer inefficiency #2 (pattern=workaround, type E) on `drive.observe`: an already-capped `drive.observe {surface:editor_chrome, window_title:BP_Button_Parent, screenshot:false, interactables_only:true, max_elements:40}` returned **61238 chars** (>10000 threshold) and spilled to `Saved/PinWright/HttpResponses/.../<uuid>.json` (`omitted_count` 236), forcing an extra `Read` for a simple toolbar-presence check. Root cause: the per-element `handle` is the full Slate widget-tree path, so `max_elements` caps count but not bytes and a capped observe still overflows. Proposed: a compact/labels-only projection, a shorter opaque handle, byte-aware truncation, or a subtree/name-filtered observe so a presence/scan check stays inline; document in `docs/wiki-src/drive.md`. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — no existing drive.observe element-list ticket; `F-drive-observe-screenshot-inline-base64` (IN-REVIEW) is the same method but the screenshot-delivery fix family and explicitly splits this out; `E-http-response-spill` (DONE) is the generic spill mechanism. Prior cross-surface element-list evidence (editor_chrome ~922KB-1.75MB; game 23898/24928 for ~26 elements; web 15344/26549; caps ineffective) lives in `F-drive-observe-screenshot-inline-base64` #2/#3. Seeded `encounters: 1` (new file); `lastSeen` set.
- `#2-reword-and-bytecap` `IN-REVIEW` developer — Reworded then fixed. REWORD: severity Medium -> **Low** (README.md:203-204 Low band = "a response spill that only forces a `Read`"; matches the whole `*-no-projection-spills` family, all Low; the Medium had borrowed a cross-surface recovery cost and an unjustified reach bump). Re-scoped OFF presence checks — a pure "is control X present?" check is served inline by `drive.expect widget_present` (fixed `{met,...}` shape, matches by `label`, cannot spill; `DriveExpectHandler.cpp:75-81`, `DriveConditionEval.h:11-14`), so the surviving gap is the **discovery/survey** observe. FIX: added `max_bytes` (default 0 = unlimited, backward-compatible) to `drive.observe` — a byte-aware cap on the elements' total serialized size, reusing `omitted_count`, applied on game/editor/web. Chose a byte cap over the family's `namesOnly`/`fields` projection because `drive.observe`'s verbose field IS its actionable-identity field (`Handle == Path`, `DriveEditorChrome.cpp:172`; actions resolve by handle, `DriveActionCommon.cpp:188-192`), so the handle can't be dropped. Files: `DriveHandlerCommon.h`/`.cpp` (new `EstimateElementJsonBytes` upper-bound helper + `max_bytes` arg in `FilterObservationElements`/`BuildObservation`), `DriveObserveHandler.cpp` (register/read/thread `max_bytes`), `DriveWebHandlers.cpp` (thread on the async web path), `Docs/wiki-src/drive.md` (document `max_bytes` + cross-ref `drive.expect`). Updated the existing `PinWright.drive.editorint.FilterObservationElements` call sites for the new arg. Regression test added: `PinWright.drive.observe.ElementListByteCapBoundsPayload` (`Tests/Drive/TestDriveObserveByteCap.cpp`) — builds an in-code synthetic observation of 60 elements with ~600-char verbose handles, asserts the uncapped `WriteObservation` serialization overflows 10000 chars, then that `max_bytes:8000` drops elements, reports `omitted_count` exactly, keeps >=1, and re-serializes within budget + root overhead (<=10000); fails if the byte cap is reverted.
