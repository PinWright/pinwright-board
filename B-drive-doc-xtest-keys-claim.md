---
id: B-drive-doc-xtest-keys-claim
title: "drive.click / drive.key docs say XTEST key events do not reach the editor's SDL window, but xdotool (XTEST) Ctrl+Z / Ctrl+Y / Ctrl+C / Ctrl+V reached a PIE game on Linux; drive.key still has no OS path"
status: IN-REVIEW
severity: Medium
category: bug
tags: [drive, drive.key, drive.click, os_input, xtest, linux, docs, hotkeys]
encounters: 1
lastSeen: 2026-09-28T10:27:00Z
---

# The "XTEST keys don't reach the editor" note is wrong, and it hides the only faithful hotkey path

`drive.click.md` (Notes, `os_input` paragraph) states: "Keys have no OS variant - XTEST key events
were observed not to reach the editor's SDL window even with X focus on it - so `drive.key` does
not accept `os_input` at all". In a visible Linux editor (UE 5.8, `/sdb-disk/src/unreal/unreal-fpv-wt1`,
plugin `61c243f5`, display `:0`, standalone PIE in the level-editor viewport) real XTEST key events
did reach the game:

```
xdotool keydown ctrl; sleep 0.3; xdotool keydown z; sleep 0.3; xdotool keyup z; sleep 0.2; xdotool keyup ctrl
```

with X focus on `PDS - Unreal Editor` fired the game's `Ctrl+Z` handler (the in-game level editor
undid a gate placement, visible in the gate list and the viewport); `Ctrl+Y`, `Ctrl+C` and `Ctrl+V`
behaved the same (redo, copy, three pastes producing gates 3/4/5). The keys must be held across at
least one engine tick; a zero-length tap was not tried.

Consequence: an agent that trusts the page concludes that faithful OS-level hotkey input is
impossible and falls back to `drive.key` (Slate injection) or console commands, which are exactly
the paths QA repro rules reject. The project memory had to carry the correction instead of the wiki.

**Fix (proposed):** correct the note (hold-duration requirement, needs X focus on the editor
window), and give `drive.key` an `os_input:true` path that does the same XTEST keydown/hold/keyup
with modifiers, gated on the focused X window belonging to the editor pid.

## History
- `#1-xtest-hotkeys-work` `OPEN` reporter - Filed from a QA verification of in-game editor undo/redo/paste in PDS (UE 5.8, Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`). Needed a hand-written xdotool helper for every hotkey because `drive.key` has no OS path and the docs said XTEST keys cannot work. Cheap once known (~5 min).
- `#2-docs-corrected` `IN-REVIEW` developer — Docs corrected to the measured behaviour; no `drive.key` OS path added (scoped out: the brief asked for the doc fix only, the `os_input:true` key path stays a follow-up on this ticket). `docs/wiki-src/drive.md`: the `os_input` limits paragraph under `drive.click` now says XTEST keys DO reach the editor, quotes the measured `xdotool keydown ctrl ... keyup ctrl` sequence, states both measured conditions (X focus on the editor window, each key held across at least one engine tick; a zero-length tap untried) and that a raw `xdotool` sequence bypasses the display lock and ownership gate; the `drive.key` section points there. The false claim was also removed from the code comments in `Source/PinWright/Private/Handlers/Drive/DriveOsInput.h` (class comment) and `Handlers/Drive/DriveActionHandlers.cpp` (comment above `DRIVE_OS_INPUT_PARAM`). Docs-only, no automation test.
- `#3-linux-verification` `IN-REVIEW` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Doc half verified on the tree: `docs/wiki-src/drive.md` (os_input limits paragraph under drive.click) now says XTEST keys do reach the editor, quotes the measured xdotool sequence and both conditions (X focus on the editor, each key held across an engine tick), and the old claim is gone from Source and docs. Not done: the ticket's proposed `drive.key {os_input:true}` path. `drive.key` still declares no `os_input` (`DriveActionHandlers.cpp` param text: 'Mouse only; drive.key has no os_input'). Stays IN-REVIEW until the owner accepts the doc-only fix and files the key path separately, or the key path ships with a visible-display test.
