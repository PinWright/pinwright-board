---
id: B-screenshot-window-title-precedence
title: "editor.screenshot_window lets the title alias silently override the canonical window_title selector"
status: OPEN
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
