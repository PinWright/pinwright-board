---
id: F-drive-observe-screenshot-inline-base64
title: "drive.observe delivers the Set-of-Mark screenshot as inline base64 only — no file-path delivery mode"
status: IN-REVIEW
severity: Medium
category: feature
tags: [drive, observe, screenshot, set-of-mark, base64, response-spill, editor_chrome, docs]
encounters: 3
lastSeen: 2026-07-03T16:30:00.0000000Z
---

# drive.observe screenshot is inline base64 only — no file-path delivery

## What's awkward
`drive.observe` delivers its Set-of-Mark PNG **only** as an inline base64 string —
there is no file/path delivery mode. The `screenshot` param is boolean
(`DriveObserveHandler.cpp:20,43`), the renderer only base64-encodes the annotated
frame with no disk write (`DriveSetOfMarkRenderer.cpp:282`), and the bytes are
serialized nested under `structuredContent.screenshot.base64`
(`DriveJson.cpp:52-62`, attached at `:204-207`), **not** top level.

A base64-encoded PNG is inherently large (~1.33× the raw bytes), so a captured
screenshot is on the order of ~100 KB of text on its own — already past the
10000-char display threshold. On the default `screenshot:true` observe the whole
response is therefore auto-dumped to disk by the `outputTooLong` mechanism, and
viewing the marked image then requires an out-of-band decode pipeline:

1. `grep` the dumped JSON for the marks/labels;
2. offset-`Read`s of the dump (one `Read` errored outright:
   "File content (431041 tokens) exceeds maximum allowed tokens");
3. hand-write PowerShell to base64-decode the blob to a `.png`.

The decode was non-obvious because the base64 lives at
`j.structuredContent.screenshot.base64`, not top level. The first two PowerShell
attempts failed ("wrote 0 bytes; marks_drawn=" — they assumed `j.screenshot.base64`)
and only the third worked after discovering "top keys: content, structuredContent,
isError". Net: a single visual-review intent (observe + eyeball a marked screenshot)
cost a grep, multiple Reads, and three PowerShell attempts on top of the RPC — pure
delivery friction, since the marked image is not available as a file the way the
other screenshot verbs already provide one.

## What it should do
Give `drive.observe` a file-delivery mode for the screenshot, mirroring how the
other screenshot verbs already return a saved `path` (`editor.screenshot` returns
`{path,width,height}` via `ScreenshotUtils::MakeScreenshotOutputPath`;
`ui.screenshot` returns `{screenshotPath,...}` under `returnBase64:false`):

- a `screenshot_mode:"file"` selector (default `"inline"`) writes the Set-of-Mark
  PNG to disk and returns `{path, width, height, marks_drawn, marks_omitted}` under
  `screenshot`, omitting the inline `base64`. The response then stays small and the
  agent can `Read` the marked image directly.

Note (correcting the original filing): the `outputTooLong` machinery does **not**
"already write big payloads to disk" on the MCP path — that server-side `/rpc`
spill is bypassed for MCP callers (see `E-http-response-spill`, DONE), and the
spill the reporter saw was the client harness's own. So the fix is a **new**
server-side write returning a path (as `editor.screenshot` already does), not a
reuse of the existing spill. Weaker alternative (docs only): at minimum document
that with `screenshot:true` the base64 lands under
`structuredContent.screenshot.base64` and give a copy-paste decode snippet.

## Scope
This ticket is **strictly the screenshot file-delivery mode**. The element-list
overflow documented in History `#2`/`#3` (the default `interactables_only`
projection itself spilling on editor_chrome/game/web, where `screenshot:false`
still exceeded the threshold) is a **separate** defect this fix does not address —
it belongs to the response-spill / `*-no-projection-spills` family (a
drive.observe element-list narrowing filter), not here. Resolving screenshot
delivery does not clear that element-list evidence; it should be filed on its own.

## Evidence
- `drive.observe {surface:editor_chrome, interactables_only:true, screenshot:true}`
  returned `outputTooLong`: "Response exceeds display limit (1094411 chars,
  threshold 10000); full payload written to ...json" (transcript line 174).
- Payload breakdown (correcting the original "the ~1MB is dominated by the base64"
  claim, which is arithmetically false): the decoded PNG was 80396 bytes, so its
  base64 is `ceil(80396/3)*4` = 107196 chars — ≈9.8% of the 1094411-char response
  (≈19.6% if counted twice for the `content[0].text` + `structuredContent`
  double-encoding). The **element-list JSON dominates** the ~1MB, not the base64.
  Routing the PNG out of band removes the ~107 KB base64 and makes the marked image
  Read-able; the residual element-list overflow is the separate concern above.
- Multi-step base64 extraction incl. the wrote-0-bytes failure at transcript
  lines 174–346; a `Read` hit the token cap; the base64 was at
  `structuredContent.screenshot.base64`, not top level.
- Call-trace source: `record.efficiency` inefficiency #2 (pattern=workaround),
  transcript
  `.../subagents/workflows/wf_60865404-cf4/agent-ae05318e07add50c7.jsonl`.

**Fix:** Add a `screenshot_mode` param ("inline" default | "file") to `drive.observe`.
In file mode, after PNG-encoding the annotated frame, write it via
`PinWrightScreenshotUtils::MakeScreenshotOutputPath` + `FFileHelper::SaveArrayToFile`
and return `screenshot.path` (with width/height/marks) instead of `screenshot.base64`.

severity rationale (screenshot delivery only, element-list spill excluded per Scope):
impact = the marked screenshot can be viewed **only** via an out-of-band decode
workaround — locate the base64 under the non-obvious nested
`structuredContent.screenshot.base64` key, then hand-write a base64-decode, because a
plain `Read` of the spilled JSON hit the 431k-token cap. That is past the Low "only
forces a `Read`" bar and meets the Medium "doable only via a documented workaround /
many extra calls" bar. reach = `screenshot:true` is the **default** for drive.observe,
so this is a normal path, not a rare one -> Medium.

## History
- `#1-initial-audit` `OPEN` reporter — Filed: `drive.observe` with `screenshot:true`
  embeds the Set-of-Mark PNG as a ~1MB inline base64 blob that overflows the
  display threshold, dumps to disk, and then can only be viewed via grep +
  offset-Reads + a hand-written PowerShell base64 decode (base64 nested under
  `structuredContent.screenshot.base64`, costing two failed extraction attempts).
  Ask: add a file-delivery mode (`screenshot:"file"` / `screenshot_path`) that
  saves the PNG and returns its path like the other screenshot verbs, so the
  element list stays small and the image is directly Read-able.
- `#2-additional-element-list-itself-spills` `OPEN` reporter — New angle on the
  same `response-spill` family from an editor_chrome find-an-asset audit (focus
  `null`, namespace `drive`, outcome `blocked_by_mcp`): here the overflow was NOT
  the screenshot base64 — it was the **element list itself**. Every `drive.observe`
  on the full editor frame (5 observes at ~922KB–1.75MB each) exceeded the
  10000-char display threshold and spilled to
  `Saved/EditorAutomation/HttpResponses/...json`, forcing repeated Grep/Read to
  locate a single `SEditableText` handle. Crucially, `interactables_only:true` and
  `max_elements` did NOT bring the payload under threshold — so this ticket's
  proposed fix (route the PNG out of band so "the element list stays small") is
  necessary but **not sufficient**: the full editor-chrome element projection is
  itself oversized. The family needs a way to narrow the element list too (a
  `handle`/subtree-rooted observe, a name/type filter, or a smaller-by-default
  projection) so locating one control does not require a grep over a 1MB+ spill.
  This is expected for a full editor frame but is real per-session navigation
  overhead on the primary editor_chrome discovery path. Call-trace source:
  transcript `.../subagents/workflows/wf_458c51e4-8a7/agent-a12452cb010faaabc.jsonl`
  (5 observe spills; friction note "interactables_only/max_elements did not bring
  it under threshold").
- `#3-additional-game-web-spill` `OPEN` reporter — Additional evidence extending
  the same element-list `response-spill` family beyond editor_chrome to
  `surface:game` AND `surface:web` on a live PIE drone-edit UI navigation via
  `drive`. `drive.observe {surface:game, interactables_only:true, screenshot:false}`
  spilled at 23898 chars for only ~26 elements (a second game observe at 24928);
  `surface:web` observes spilled at 15344 and 26549 chars. Confirms `#2`:
  `interactables_only:true` did NOT bring it under the 10000-char threshold —
  long nested UMG handle paths (e.g.
  `W_OverallUILayout_C_0/W_MyDrones_C_0/W_DroneSelectionButton_C_8/CustomiseButton/SCommonButton`)
  dominate the payload. New angle: extraction went beyond a plain `Read` — the
  dumped JSON had to be Grep/Read'd and parsed with an external Python script,
  which hit a `cp1251` `UnicodeDecodeError` on Cyrillic (Russian) element labels
  and forced an explicit utf-8 open, so the spill also has a non-ASCII-label
  encoding-fragility angle. Bumped severity Low->Medium: the spill now reproduces
  on the DEFAULT (interactables_only, no-screenshot) element projection across
  editor_chrome + game + web — an every-session drive navigation path (reach
  bump) — and recovery required an out-of-band script plus a utf-8 fix, past the
  "only forces a Read" bar for Low. Call-trace source: this session's live PIE
  drone-edit drive navigation.
- `#4-reworded-and-file-delivery-implemented` `IN-REVIEW` developer — Reworded to
  match source + reality, then implemented. Rescope: title/body/Fix retargeted to
  **strictly** the screenshot file-delivery ask; corrected two false claims — (a)
  the base64 does NOT dominate the ~1MB (80396 raw bytes -> 107196 base64 chars ≈
  9.8% of the 1094411-char response; the element list dominates), and (b)
  `outputTooLong` is bypassed for MCP callers so the fix is a NEW server-side write,
  not a reuse of that spill; re-anchored the severity rationale on the
  screenshot-decode workaround alone (kept Medium) and removed the self-contradicting
  "-> Low" line; added a Scope section splitting the element-list-projection overflow
  (History #2/#3) out to the response-spill/`*-no-projection-spills` family.
  Implemented `screenshot_mode` ("inline" default | "file") on `drive.observe`: file
  mode PNG-encodes then writes to `Saved/Screenshots/Drive` via
  `PinWrightScreenshotUtils::MakeScreenshotOutputPath` + `FFileHelper::SaveArrayToFile`
  and returns `screenshot.path` (with width/height/marks) while omitting `base64`;
  inline mode unchanged. Files: `Handlers/Drive/DriveTypes.h` (add `FDriveScreenshot::Path`),
  `Handlers/Drive/DriveSetOfMarkRenderer.h/.cpp` (new `DeliverScreenshotBytes` seam +
  `bWriteToFile` on `CaptureAnnotated`), `Handlers/Drive/DriveJson.cpp` (`WriteScreenshot`
  emits `path` xor `base64`), `Handlers/Drive/DriveHandlerCommon.h/.cpp`
  (`BuildObservation` `bScreenshotToFile` passthrough), `Handlers/Drive/DriveObserveHandler.cpp`
  + `Handlers/Drive/DriveWebHandlers.cpp` (parse `screenshot_mode`, thread through
  game/editor_chrome + web). Regression test:
  `PinWright.drive.somrender.FileDeliveryReturnsPathNotBase64`
  (`Tests/Drive/TestDriveScreenshotFileDelivery.cpp`) — asserts file mode writes the
  bytes to a path with base64 omitted (and JSON emits `path`, not `base64`) while
  inline mode keeps base64 and emits no path; fails if the fix is reverted.
