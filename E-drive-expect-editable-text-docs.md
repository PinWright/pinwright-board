---
id: E-drive-expect-editable-text-docs
title: "drive.md wiki does not document editable-input read-back — observe now surfaces an editable input's live typed value in a distinct `value` field and text_equals/text_contains verify it (fallback to label), but drive.md mentions none of it"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [drive, editor_chrome, editable-text-readback, value-field, text-verification, docs]
encounters: 1
lastSeen: 2026-07-02T12:22:54.0000000Z
---

# drive wiki does not document editable-input read-back (the `value` field and value-preferring text conditions)

## What's missing
On the `editor_chrome` (and `game`) surface, `drive.observe` now surfaces an
editable input's **live typed value** in a distinct `value` field, and
`drive.expect`/`drive.wait_for` `text_equals`/`text_contains` compare that `value`
when present (falling back to `label`). This is the read-back path the companion
code fix `B-drive-editable-text-reads-hint-not-value` added, and it is already
live in the plugin source:

- `DriveEditorChrome.cpp:156` / `DriveLiveResolver.cpp:330` set
  `Element.Value = DriveSlateTextValue::ReadEditableWidgetText(SlateWidget)`, which
  reads the live `GetText()` of `SEditableText` / `SEditableTextBox` /
  `SMultiLineEditableText` / `SMultiLineEditableTextBox` (`DriveSlateTextValue.h:37-59`).
- `DriveConditionEval.cpp:68-75` (`DriveTextComparand`) prefers `Element.Value` when
  non-empty; `text_equals` (`:214-219`) / `text_contains` (`:237-242`) compare it and
  report `actual="value=\"...\""`.
- `DriveJson.cpp:96-101` emits a distinct per-element `value` (omitted when empty);
  the backing field is `FDriveElement.Value` (`DriveTypes.h:39`).

So `drive.type "Cube"` into a search box, then
`drive.expect {type:text_equals, expected_text:"Cube"}` now returns `met:true` —
the round-trip the original audit thought impossible.

But `docs/wiki-src/drive.md` documents **none** of this (ripgrep for
`value|hint|placeholder|typed|editable|read.?back` returns zero hits):
- The observe element-field list (`drive.md:17`) names `handle`, `type`, `label`,
  the flags, `geometry.absolute`, `surface` — but not `value`.
- The condition list (`drive.md:42`) is the bare
  `` `text_equals` / `text_contains` — `target` + `expected_text`. `` with no note
  that these compare the live typed `value` (falling back to `label`).
- `drive.type` (`drive.md:70`) never says the text it enters is observable
  afterwards as `value`.

Consequence: an agent that types into an editable box has no doc telling it the
typed text is now read back as a distinct `value` field (with `label` still holding
that widget's hint/placeholder), so it cannot discover the supported
`drive.type` -> `drive.expect text_*` verification pattern from the wiki.

## What it should say
`docs/wiki-src/drive.md` should, near the `observe` element-field list, the
`text_equals`/`text_contains` condition list, and the `drive.type` section:
- state that an editable input's **live typed text** is surfaced by `observe` in a
  distinct `value` field (present only when non-empty), separate from `label`,
  which for an editable input holds its accessible/**hint** text;
- state that `text_equals`/`text_contains` compare the element's `value` when present
  (falling back to `label`), so after `drive.type` you **can** assert the text
  actually entered into a search/edit box, and the failure `actual` names which
  field it read (`value="..."` vs `label="..."`).

So an agent can discover the type-then-verify read-back from the wiki instead of
diving into C++.

## Evidence
- Code chain confirmed at plugin HEAD `561ed32` (the B-fix commit `79656cc` is a
  confirmed ancestor): `DriveSlateTextValue.h:37-59`, `DriveEditorChrome.cpp:156`,
  `DriveLiveResolver.cpp:330`, `DriveConditionEval.cpp:68-75` / `:214-219` / `:237-242`,
  `DriveJson.cpp:96-101`, `DriveTypes.h:39`.
- Docs gap confirmed: ripgrep of `docs/wiki-src/drive.md` for
  `value|hint|placeholder|typed|editable|read.?back` returns zero hits;
  `drive.md:17` / `:42` / `:62` / `:70` never mention the `value` field.
- Original audit call-trace: `record.efficiency` inefficiency #2 (pattern=wiki-nav,
  method `drive.observe`), transcript
  `.../subagents/workflows/wf_458c51e4-8a7/agent-a12452cb010faaabc.jsonl`.

## Distinct from
- `B-drive-editable-text-reads-hint-not-value` (IN-REVIEW) — the **code fix** that
  added the `value` read-back (already an ancestor of plugin HEAD, with its own
  regression test `Source/PinWright/Private/Tests/Drive/TestDriveEditableTextValue.cpp`).
  This ticket is the orthogonal **docs** follow-up: commit `79656cc` did not touch
  `docs/wiki-src/drive.md`, so the wiki still doesn't tell agents the read-back exists.

severity rationale: impact=minor completeness gap — an undocumented `value` field and
value-preferring text conditions; the capability works, it just isn't in the wiki, so
no data is lost/lied about and no source dive is forced now that the code round-trips ×
reach=the `drive.type` -> `drive.expect text_*` path is a common editor_chrome flow ->
Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of an editor_chrome
  find-an-asset task (focus `null`, namespace `drive`, outcome **blocked_by_mcp**).
  PROCESS/discoverability friction: `docs/wiki-src/drive.md` documents
  `text_equals`/`text_contains` (line 42) with no caveat that on `editor_chrome`
  they compare an editable input's placeholder/label, never the typed value, and
  neither `observe` nor `type` docs note that a typed value is not surfaced as
  `label`. Having read all relevant drive wiki pages, the agent still had to
  reverse-engineer the plugin C++ (`DriveConditionEval.cpp`, `DriveEditorChrome.cpp`)
  and installed-engine source (`SEditableText.cpp`, `SWidget.cpp`) to discover the
  read-back mechanism — the agent's own friction note calls that C++ dive "itself a
  discoverability gap". Ask: document in `drive.md` that editable-input text reads
  the accessible/hint text (not the live `GetText()` value), so `text_equals`/
  `text_contains` cannot assert a typed value, and point to the supported way to
  verify (or to verifying the filter's effect instead). Deduped against
  `B-drive-editable-text-reads-hint-not-value` (the code fix) — filed separately as
  the orthogonal docs angle.
- `#2-reword-and-docs` `IN-REVIEW` developer — REWORD + docs implementation. The
  ticket's premise (an editable input's typed value is unreadable; `text_equals`/
  `text_contains` compare the placeholder/label) is FALSE at plugin HEAD `561ed32`:
  the companion code fix `B-drive-editable-text-reads-hint-not-value` (commit
  `79656cc`, a confirmed ancestor) already landed the `value` read-back
  (`DriveSlateTextValue.h:37-59`, `DriveEditorChrome.cpp:156`, `DriveLiveResolver.cpp:330`,
  `DriveConditionEval.cpp:68-75`/`:214-219`/`:237-242`, `DriveJson.cpp:96-101`,
  `DriveTypes.h:39`), so `drive.type` -> `drive.expect text_equals` now returns
  `met:true`. Documenting the old "readback impossible / verify the effect instead"
  caveat would ship a falsehood, so inverted the ticket to the real surviving gap:
  `drive.md` never mentions the `value` field or the value-preferring text conditions
  (rg for `value|hint|editable` = 0 hits). Fix (docs-only) in
  `Plugins/PinWright/Docs/wiki-src/drive.md`: documented that observe surfaces an
  editable input's live typed value in a distinct `value` field (separate from
  `label`=hint), and that `text_equals`/`text_contains` compare `value` when present
  (fallback to `label`) so `drive.type` -> `drive.expect text_*` verifies what was
  typed — added to the `## Observation` element-field list (line 17), the
  `## Verification` condition list (line 42), the `### drive.observe` return
  description, and the `### drive.type` section. Severity Medium -> Low (completeness
  gap, not a blocker). No code/test change — docs-only; the production behavior is
  already covered by B's `TestDriveEditableTextValue.cpp`. Plugin compiles clean.
