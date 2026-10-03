---
id: B-screenshot-window-title-precedence
title: "editor.screenshot_window lets the title alias silently override the canonical window_title selector"
status: DONE
severity: Medium
category: bug
tags: [editor, screenshot, window-selector, alias, precedence]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# The screenshot title alias overrides the canonical selector

## What happens

The shared selector parser resolves the title with
`GetStringFirstOf({"title", "window_title"})`, placing the alias first
(`Source/PinWright/Private/Handlers/Drive/DriveHandlerCommon.cpp:130-137`).
`editor.screenshot_window` registers `window_title` as the canonical parameter and
documents `title` as its alias
(`Handlers/Editor/EditorWindowHandlers.cpp:995-1007`). If a caller supplies both
with different values, the alias silently wins.

## Why it matters

A successful screenshot can target the wrong editor window while the caller trusts
the canonical field it supplied. Severity is Medium: the result is silently wrong,
but the conflict occurs only when both equivalent fields are present and has a simple
workaround.

## What should happen

Resolve `window_title` before `title`, or reject conflicting canonical and alias
values with a typed ambiguity error. Add coverage for equal and conflicting pairs,
and apply the same declared precedence consistently to shared window selectors.

## Workaround

Supply only `window_title` or only `title`, never both.

## Related

- `B-screenshot-window-default-selector-captures-foreign-window` — wave-6 ticket
  whose selector fix review exposed the alias-precedence defect.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed alias-first parsing at `DriveHandlerCommon.cpp:130-137` against the canonical-first registration vocabulary at `EditorWindowHandlers.cpp:995-1007`. No screenshot, build, test, editor, or MCP call was run. Severity Medium because the wrong-target result requires contradictory duplicate fields and supplying one selector is a workaround.
- `#2-canonical-first-selector` `IN-REVIEW` developer — Still reproducible before the fix: `FDriveHandlerCommon::ParseWindowSelector` read `GetStringFirstOf({title, window_title})` (alias first) while the index pair already read `{window_index, index}` (canonical first). Swapped the title pair to `{window_title, title}` in `Source/PinWright/Private/Handlers/Drive/DriveHandlerCommon.cpp`, so the canonical key wins in both pairs for every caller of the shared parser (`editor.screenshot_window`, the other `editor.*` window verbs at `EditorWindowHandlers.cpp`, `drive.observe` / `drive.expect` / the action verbs on `editor_chrome`). Empty `window_title` still falls through to `title` (GetStringFirstOf skips empty strings), matching the documented "non-empty title" rule. Chose canonical-first over a typed ambiguity error: it is the same rule the index pair already follows and does not add a new refusal. The `title` alias descriptions (`DRIVE_WINDOW_SELECTOR_PARAMS` in `DriveHandlerCommon.h` and the `editor.screenshot_window` registration) now say `window_title wins when both are non-empty`; wiki-src `drive.md` (Surfaces) and `editor.md` (`editor.screenshot_window`) state the precedence; CHANGELOG notes the behaviour change. Test `PinWright.editor.screenshot_window.CanonicalSelectorKeysWinOverAliases` in `Tests/EditorOps/TestEditorScreenshotWindowSelector.cpp` (pure parse, no Slate) covers conflicting and equal `window_title`/`title` and `window_index`/`index` pairs plus empty-canonical fallthrough; the conflicting-title assertion fails if the alias-first order is restored.
- `#3-verified-linux` `DONE` tester — Verified on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d). run3/full: `PinWright.editor.screenshot_window.CanonicalSelectorKeysWinOverAliases` passed, non-skipped. With conflicting `window_title`/`title`, the canonical key wins, and the same holds for `window_index`/`index`. Equal pairs resolve the same, and an empty canonical value falls through to the alias. The fix is in the shared `FDriveHandlerCommon::ParseWindowSelector`, so every caller (editor.* window verbs, drive.* on editor_chrome) gets the same declared precedence, as the ticket asked. Doc checked: the `title` param text in `DriveHandlerCommon.h` (DRIVE_WINDOW_SELECTOR_PARAMS) says "window_title wins when both are non-empty". The CHANGELOG notes the behaviour change. Canonical-first was chosen over an ambiguity error; the ticket offered either.
