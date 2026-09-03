---
id: B-screenshot-window-default-selector-captures-foreign-window
title: "editor.screenshot_window with no window selector silently captures whatever window is active — in a shared editor that is another agent's asset editor"
status: OPEN
severity: High
category: bug
tags: [screenshot, screenshot_window, shared-editor, silent-wrong, evidence, window-selector, visual-review]
---

# `editor.screenshot_window` with no selector silently captures another agent's window

`editor.screenshot_window`'s window selector is optional: "Both selectors unset picks the **active**
top-level window." In a single shared editor with six agents working concurrently, "the active
top-level window" is **not** a property the caller controls — any other agent opening an asset
editor changes it between two of your calls. The verb then returns a clean success for a PNG of
somebody else's screen.

## Repro (measured 2026-09-03T00:29Z, EAContentExamples58, port 27145)

A capture series of the PIE viewport, every call identical except the filename:

- shots 1-5 → `{"width":1927,"height":1087,"windowTitle":"EAContentExamples58 - Unreal Editor"}` — correct.
- shot 6, same args → `{"width":5120,"height":1386,"windowTitle":"SM_WPN_AR","sizeBytes":3896232}`

Between shot 5 and shot 6 another agent opened the `SM_WPN_AR` Static Mesh editor. My call captured
**their** window, at their size, and reported success. Re-issuing the identical call with
`window_title: "EAContentExamples58 - Unreal Editor"` returned 1927x1087 again.

## Why this is worse than an ergonomic wart

The result is **silently wrong evidence**, the failure class this board treats as most serious. The
response is a success with a valid PNG and plausible metadata; nothing says "this is not the surface
you have been capturing". A reviewer who does not diff `windowTitle` across a series — and there is
no reason to expect it to change, since the args did not — files a frame of an unrelated asset
editor as proof of a HUD. Two of the fields that would betray it (`width`/`height`) only differ
because the other agent's window happened to be a different size; a same-size window would be
undetectable from the response alone.

It also silently spends a capture slot: in a 10-minute world-lock window with PIE running, the shot
cannot be retaken later because the transient state (a paused frame, a fading HUD animation) is gone.

## What is wanted

Any of, in preference order:

1. **Make the selector required** for `screenshot_window`, or default it to the **main frame** window
   rather than the active one. A caller wanting "whatever is focused" can ask for it explicitly.
2. Failing that, return the resolved selection prominently and add a `selectorWasDefaulted: true`
   field so a caller can assert on it, and document the shared-editor hazard on the method page.
3. Optionally accept `window_title` on the first call of a session and remember it.

`windowTitle` is already returned, which is the right raw material — the defect is that the default
makes it a *result* instead of an *input*.

## Workaround

Always pass `window_title` (or `window_index`), never rely on the default, and assert the returned
`windowTitle` matches what you asked for. Related but distinct: `B-no-way-to-capture-pie-pixels`
(no PIE-specific capture surface) and `F-pie-capture-fixed-size-with-umg` (no caller-specified
resolution).

## History
- `#1-filed` `OPEN` reporter — Filed while capturing `/Game/FPS/UI/WBP_HUD` for a critic review in a
  six-agent shared editor. Five identical calls captured the main editor window; the sixth captured
  another agent's `SM_WPN_AR` asset editor at 5120x1386 and reported success. Caught only because the
  returned `width`/`height` changed; a same-size foreign window would have passed as evidence.
