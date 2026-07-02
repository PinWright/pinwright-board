---
id: B-drive-editable-text-reads-hint-not-value
title: "drive editor-chrome reads an editable field's HINT text as its Label, so observe/expect never surface the typed value (text_equals/text_contains silently false)"
status: IN-REVIEW
severity: High
category: bug
tags: [drive, editor_chrome, editable-text-reads-hint, text-verification, readback, extract-label]
encounters: 1
lastSeen: 2026-07-02T15:21:30.3411737+03:00
---

# drive editor-chrome editable-text value reads the hint, never the typed text

## What's wrong
On the `editor_chrome` surface, `drive.observe`/`drive.expect` compute an
element's `Label` via `DriveEditorChrome::ExtractLabel`, whose first (and, for
input widgets, only) branch is `SWidget::GetAccessibleText(EAccessibleType::Main)`.
For editable-input widgets (`SEditableText`, and therefore the
`SFilterSearchBox`/`SAssetSearchBox`/`SSearchBox` that wrap one) the engine
resolves that accessible text to the widget's **hint/placeholder text**, not its
live typed value — `SEditableText::GetDefaultAccessibleText` returns
`GetHintText()`, never `GetText()`.

Consequence: after `drive.type` puts "Cube" into the Content Browser search box,
`drive.expect {type:text_equals|text_contains, target:<the SEditableText handle>,
expected_text:"Cube"}` returns `met:false` with `actual="label=\"Search Assets\""`
(the hint). There is NO branch in `ExtractLabel` that reads `GetText()` for
editable widgets, so the live value is never surfaced by `observe` and never
available to the `text_equals`/`text_contains` conditions. The canonical
type-then-verify editor-chrome workflow (`drive.type` -> `drive.expect text_*`)
is therefore structurally impossible for any editable field, and worse, the
result is a **silent lie**: the caller who typed "Cube" is told the field holds
"Search Assets", which reads as "the type didn't land."

## What it should do
`ExtractLabel` (or a dedicated value-read path for input widgets) should surface
an editable widget's **current text** so `observe`/`expect` can verify it:
- Add a `GetText()` branch for the editable Slate types before the accessible-text
  fallback — e.g. `SEditableText::GetText()`, `SMultiLineEditableText::GetText()`,
  `SEditableTextBox::GetText()`, and unwrap `SSearchBox`/`SFilterSearchBox`/
  `SAssetSearchBox` to their inner editable text. Only fall back to the hint
  (accessible text) when the value is empty, or expose value and hint separately
  (e.g. a `value` field distinct from `label`) so callers can tell an empty field
  from one holding the hint.
Then `text_equals`/`text_contains` against a search/input field reflect what was
typed instead of the placeholder.

## Verbatim repro
Content Browser drawer open; the asset search `SEditableText` handle observed at
`.../SAssetSearchBox[0]/SMenuAnchor[0]/SFilterSearchBox[0]/SMenuAnchor[0]/SFilterSearchBoxImpl[0]/SHorizontalBox[0]/SBox[3]/SEditableText[0]`.
1. `mcp__pinwright__call` method=`drive.type`
   args=`{surface:editor_chrome, window_index:0, handle:"<that SEditableText handle>", text:"Cube"}`
   -> success (`{outcome, changed, settled, condition_met, ...}`).
2. `mcp__pinwright__call` method=`drive.expect`
   args=`{surface:editor_chrome, window_index:0, condition:{type:"text_equals", target:"<that SEditableText handle>", expected_text:"Cube"}}`
   -> `{"met":false,"actual":"label=\"Search Assets\"","expected":"label==\"Cube\"","detail":"element handle='...SEditableText[0]'","root_name":"EAContentExamples57 - Unreal Editor"}`
3. Same with `condition.type:"text_contains"` ->
   `{"met":false,"actual":"label=\"Search Assets\"","expected":"label contains \"Cube\"",...}`.
Both conditions report the hint ("Search Assets"), never the typed "Cube".

## Guilty source (ground truth, read verbatim)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveEditorChrome.cpp:44-80`
  `ExtractLabel` — first branch (line 49):
  `const FText AccessibleText = Widget->GetAccessibleText(EAccessibleType::Main);`
  returned as the `Label` when non-empty. The remaining branches are STextBlock
  content (63-71), the widget tag (73-77), and the type string (79) — **none reads
  `GetText()` on an editable widget**. `MakeElement` sets
  `Element.Label = ExtractLabel(SlateWidget);` (line 145).
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveConditionEval.cpp:194`
  (TextEquals): `Result.bMet = Match->Label.Equals(Condition.ExpectedText, ESearchCase::CaseSensitive);`
  and line 195: `Result.Actual = FString::Printf(TEXT("label=\"%s\""), *Match->Label);`
  — the condition compares the ExpectedText against `Label` only.
- `DriveConditionEval.cpp:213` (TextContains): `Result.bMet = Match->Label.Contains(Condition.ExpectedText, ESearchCase::CaseSensitive);`
- Engine confirmation (why the accessible text is the hint):
  `C:/UE_5.7/Engine/Source/Runtime/Slate/Private/Widgets/Input/SEditableText.cpp:704-707`:
  `TOptional<FText> SEditableText::GetDefaultAccessibleText(EAccessibleType AccessibleType) const { return GetHintText(); }`
  while the live value lives at `SEditableText.cpp:92-95` (`GetText()` -> `EditableTextLayout->GetText()`).
  `C:/UE_5.7/Engine/Source/Runtime/SlateCore/Private/Widgets/SWidget.cpp:2129-2133`
  shows `GetAccessibleText` falling through to `GetDefaultAccessibleText` in the
  default/Auto behavior, so the hint is what surfaces.

severity rationale: impact=silent wrong/stale data on a normal path (the caller
who typed a value is told the field holds the placeholder, and text_equals/
text_contains silently fail — a lie the caller trusts) × reach=drive.type ->
drive.expect text verification of an input field is an every-session editor-chrome
path -> High

## History
- `#1-initial-repro` `OPEN` reporter — Filed: editor-chrome `drive.observe`/
  `drive.expect` read an editable widget's `Label` from
  `ExtractLabel -> GetAccessibleText`, which for `SEditableText` (and the
  search-box wrappers) resolves to the HINT text, never `GetText()`. Live repro:
  `drive.type "Cube"` into the Content Browser search `SEditableText`, then
  `drive.expect {text_equals|text_contains, expected_text:"Cube"}` both return
  `met:false, actual=label="Search Assets"` (the placeholder). No `GetText()`
  branch exists in `ExtractLabel` for editable inputs, so the typed value is
  unverifiable via observe/expect and the reported `actual` is a silent lie.
  Guilty lines: `DriveEditorChrome.cpp:44-80` (ExtractLabel, accessible-text-first),
  `DriveConditionEval.cpp:194 & 213` (compare against `Label`); engine root cause
  `SEditableText.cpp:704-707` (`GetDefaultAccessibleText` returns `GetHintText()`).
- `#2-fix` `IN-REVIEW` developer — Root-cause fix (both ExtractLabel copies + the text
  conditions). Added a new header-only reader `DriveSlateTextValue::ReadEditableWidgetText`
  (`Handlers/Drive/DriveSlateTextValue.h`) that reads an editable widget's LIVE typed value via
  `GetText()` for the four base Slate editable leaf types (`SEditableText`,
  `SMultiLineEditableText`, `SEditableTextBox`, `SMultiLineEditableTextBox`); search-box wrappers
  are covered because the walk emits their inner `SEditableText` as its own element. Added a
  distinct `Value` field to `FDriveElement` (`DriveTypes.h`) rather than overloading `Label`, so
  an empty field is distinguishable from a hint-only one. Populated `Element.Value` in BOTH
  `MakeElement` copies — editor_chrome (`DriveEditorChrome.cpp`) AND the game/UMG surface
  (`DriveLiveResolver.cpp`), fixing the same latent bug on `surface:game`. `text_equals`/
  `text_contains` (`DriveConditionEval.cpp`) now prefer `Value` when present and report
  `actual="value=\"...\""`, falling back to `Label` (`label="..."`) when there is no value, so
  existing static-label checks are unchanged. Surfaced `value` in the observe JSON
  (`DriveJson.cpp WriteElement`, omitted when empty). Regression test
  `Source/PinWright/Private/Tests/Drive/TestDriveEditableTextValue.cpp` (two automation tests:
  `PinWright.drive.editabletext.ReadValueNotHint` builds real in-code editable widgets with a
  HINT set and asserts the reader returns the typed value not the hint;
  `PinWright.drive.editabletext.ConditionPrefersValue` asserts text_equals/text_contains match
  the typed value and report `value="..."`, and that a value-less element still matches on label)
  — both fail if the fix is reverted. Fixture is built entirely in code (no content asset). Plugin
  compiles clean.
