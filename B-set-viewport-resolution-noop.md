---
id: B-set-viewport-resolution-noop
title: "misc.set_viewport_resolution is a silent no-op that echoes its input (never calls r.SetRes)"
status: IN-REVIEW
severity: High
category: bug
tags: [misc, viewport, screenshot, silent-success, no-op, resolution]
---

# misc.set_viewport_resolution is a silent no-op that echoes its input (never calls r.SetRes)

`misc.set_viewport_resolution` returns a clean success and echoes back the
requested `width`/`height` plus the note `"Viewport resolution preferences
set."`, but it does **nothing** — it never changes any viewport resolution.
The handler's own description and wiki claim it "Force[s] a specific
resolution on the active editor viewport via 'r.SetRes'", yet the handler body
contains no `r.SetRes` call (no `IConsoleManager`, no `GEngine->Exec`, no
`r.SetRes` string anywhere in the plugin source). The entire effect of the
`#if MCP_HAS_LEVEL_EDITOR` block is a single `UE_LOG` line; then it builds a
response that just re-serializes the input `Width`/`Height` and sends success
(`MiscHandler.cpp:259-290`).

Because the response field `width`/`height` is the *input echoed back* (not a
read-back of the actual viewport/render resolution), a caller cannot tell the
call did nothing. This is concretely misleading: a press-kit / hero-shot
workflow set 2560x1440 here, saw `{"width":2560,"height":1440,...}` echoed,
then `editor.screenshot` captured at the level-editor viewport's real native
size (957x231) — the resolution seed never drove capture size, and the
confident `2560x1440` echo masked that. The method silently succeeds with no
effect, which is the canonical "silent success-with-no-effect" tool bug.

This is the seed method's own defect, not a screenshot bug:
`editor.screenshot` correctly captures the level viewport at native resolution
(documented; see `F-editor-viewport-screenshot`). The disconnect is that
`set_viewport_resolution` advertises (description + wiki + response note) a
capability — forcing capture/output size — that it does not implement at all.

**Workaround:** none through this RPC. For a target-size capture, drive a
real resolution change yourself (e.g. `editor.console_command
{command:"HighResShot 2560x1440 filename=\"x\""}` writes to
`Saved/Screenshots/WindowsEditor/x.png`), then read the PNG off disk.

**Fix:** either (a) actually apply the requested resolution — issue the
`r.SetRes <W>x<H>` console command (or resize the capture target the
screenshot path reads) and have `editor.screenshot` honor it — and return a
true read-back of the applied size; or (b) if forcing editor-viewport
capture size is out of scope, stop claiming it: drop the `r.SetRes`/"preferences
set" language and return an explicit error or a clearly-labeled "requested
(not applied)" result so callers don't trust the echo. Whichever path, the
description/wiki must match the implementation.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live against
  `mcp__editor-automation__call`. `misc.set_viewport_resolution {width:2560,
  height:1440}` → success `{"width":2560,"height":1440,"note":"Viewport
  resolution preferences set. Actual resolution depends on editor window
  size."}`. Immediately after, `editor.screenshot {filename:"replay_setres_probe.png"}`
  → job `j_20260619T004555_a70b51c8`; `system.job_status` →
  `{"status":"completed","result":{"width":957,"height":231,...}}` — the
  capture ignored the 2560x1440 request entirely. Source confirms the
  handler never calls `r.SetRes` (only a `UE_LOG`, then it echoes the input
  width/height back as success) at `MiscHandler.cpp:259-290`; a source-wide
  grep for `r.SetRes`/`SetResolution` finds no match. So the success +
  `2560x1440` echo + `"preferences set"` note are all unbacked: the call is a
  silent no-op.
- `#2-fail-loud-not-implemented` `IN-REVIEW` developer — Applied Fix option
  (b), the accepted silent-success-class resolution (cf.
  `B-material-stub-handlers-silent-success` DONE,
  `B-input-trigger-modifier-stub-silent-success` IN-REVIEW). Option (a) was
  rejected: `r.SetRes` only resizes the game/PIE window, not the Level Editor
  viewport (which renders at its variable Slate window size), so it cannot
  deliver the advertised behavior. In
  `Handlers/Utility/MiscHandler.cpp` the no-op body (the `#if MCP_HAS_LEVEL_EDITOR`
  `UE_LOG`, then the input-echoed `SendSuccess` with the "Viewport resolution
  preferences set." note) is replaced with
  `SendError("NOT_IMPLEMENTED", …)` whose message points callers at the verbs
  that actually capture at an exact size: `render.capture_open_level {width,
  height}` and `editor.console_command {command:"HighResShot WxH"}`. The
  `REGISTER_RPC_HANDLER` description (which had falsely claimed it forces a
  resolution "via 'r.SetRes'") now states it returns NOT_IMPLEMENTED and names
  the alternatives, so the auto-generated wiki regenerated at editor launch
  matches the implementation (no `set_viewport_resolution` entry exists under
  `docs/wiki-src/`, so the registry description was the only contract surface).
  Regression test: in `Private/Tests/Utility/TestUtilityHandlers.cpp` the old
  no-crash `FMiscSetViewportResolutionValidParamsTest` is replaced by
  `FMiscSetViewportResolutionReturnsNotImplementedTest`, which invokes the real
  handler via `InvokeHandlerWithCapture` and asserts `bSuccess == false` and
  `ErrorCode == "NOT_IMPLEMENTED"` — it would fail if the input-echoing
  silent-success stub were restored. Not compiled/run here (later phase
  verifies).
