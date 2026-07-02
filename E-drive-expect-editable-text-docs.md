---
id: E-drive-expect-editable-text-docs
title: "drive.md wiki omits that editor_chrome editable-input text is NOT read back — text_equals/text_contains compare the placeholder/label, forcing a C++ dive to learn a typed value can't be asserted"
status: OPEN
severity: Medium
category: ergonomic
tags: [drive, editor_chrome, editable-text-reads-hint, text-verification, docs]
encounters: 1
lastSeen: 2026-07-02T12:22:54.0000000Z
---

# drive wiki does not document that an editable input's typed value is unreadable via observe/expect

## What's unclear
On the `editor_chrome` surface, `drive.observe`/`drive.expect` compute an
element's text from `ExtractLabel`, which for editable-input widgets
(`SEditableText`, `SSearchBox`, `SFilterSearchBox`, `SAssetSearchBox`) resolves
to the widget's **hint/placeholder** text, not its live typed value (the code
defect is tracked separately in `B-drive-editable-text-reads-hint-not-value`).
The wiki (`docs/wiki-src/drive.md`) does not mention this at all:

- `drive.md:42` documents the conditions as
  `` `text_equals` / `text_contains` — `target` + `expected_text`. `` with **no
  caveat** that for an editable box these compare against the placeholder/label,
  never the value the caller just typed.
- There is no note on `drive.observe` or `drive.type` that a typed value is not
  surfaced as `label`, and no pointer to a supported way (if any) to read an
  input's current text back.

Consequence: the canonical `drive.type` -> `drive.expect text_*` verify pattern
reads plausibly correct from the docs, so the agent tried it, got
`met:false actual=label="Search Assets"`, and then — because nothing in the wiki
explains the read-back behavior — had to **reverse-engineer the plugin and engine
C++** (`DriveConditionEval.cpp`, `DriveEditorChrome.cpp`, engine
`SEditableText.cpp`/`SWidget.cpp`) to learn that the typed value is simply not
observable. That C++ dive is pure discoverability cost that a one-line wiki caveat
would have avoided.

## What it should say
`docs/wiki-src/drive.md` (the single drive overlay; there are no per-method
observe/expect/type overlays) should, near the `text_equals`/`text_contains`
condition list and the `observe`/`type` sections:
- state that Slate text is read via `ExtractLabel` — `STextBlock` content via
  `GetText`, but **editable inputs surface their accessible/hint text, not the
  live typed value** — so `text_equals`/`text_contains` **cannot assert a value
  typed into a search/edit box** in `editor_chrome`;
- point to the supported way to verify a typed value (if/when
  `B-drive-editable-text-reads-hint-not-value` adds a `GetText()` branch or a
  distinct `value` attribute) — or, until then, say to verify the *effect* of the
  filter (e.g. the resulting asset grid) rather than the field contents.

So an agent does not need to dive into C++ to learn it can't read back what it
typed.

## Evidence
- The agent read the relevant drive wiki pages (transcript observe read line 100,
  expect line 141, type line 118) and still had to reverse-engineer the C++
  (SAY line 372: "let me look at the plugin's drive/expect source to see how it
  extracts text"; SAY line 622 concludes "the editor-chrome verification path
  cannot read the live text value of an editable input widget"). The final
  friction note calls the C++ dive "itself a discoverability gap".
- Guilty read-back path (for the doc author's context, code fix is
  `B-drive-editable-text-reads-hint-not-value`): `DriveConditionEval.cpp:194/213`
  compare against `Label`; `DriveEditorChrome.cpp` `ExtractLabel` has no editable
  `GetText()` branch; engine `SEditableText.cpp:704-706`
  `GetDefaultAccessibleText` returns `GetHintText()`.
- Call-trace source: `record.efficiency` inefficiency #2 (pattern=wiki-nav,
  method `drive.observe`), transcript
  `.../subagents/workflows/wf_458c51e4-8a7/agent-a12452cb010faaabc.jsonl`.

## Distinct from
- `B-drive-editable-text-reads-hint-not-value` (OPEN, filed by the judge) — that is
  the **code fix** (add a `GetText()` branch / expose the value). This ticket is the
  orthogonal **docs** gap: even before the code is fixed, the wiki should warn the
  caller that the round-trip is not supported, so the C++ dive is avoided. Downstream
  the wiki edit is a separate process from the code fix.

severity rationale: impact=a blocker with a workaround — the docs silence forces a
plugin+engine source dive to learn the read-back limitation (a genuine per-session
cost on the editor_chrome type-then-verify path), but no data is lost or lied about
(that lie is the B-ticket) × reach=drive.type -> drive.expect text verification is an
every-session editor_chrome path -> Medium

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
