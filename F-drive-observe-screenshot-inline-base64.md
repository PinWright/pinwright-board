---
id: F-drive-observe-screenshot-inline-base64
title: "drive.observe returns the Set-of-Mark screenshot as a ~1MB inline base64 blob with no file-path delivery option"
status: OPEN
severity: Medium
category: feature
tags: [drive, observe, screenshot, set-of-mark, base64, response-spill, editor_chrome, docs]
encounters: 3
lastSeen: 2026-07-03T16:30:00.0000000Z
---

# drive.observe screenshot is inline base64 only — no file-path delivery

## What's awkward
When `drive.observe` is called with `screenshot:true`, the Set-of-Mark PNG is
embedded in the response as an **inline base64 string**. On a real editor-chrome
observe this produces a ~1MB payload that blows past the display threshold and
gets auto-dumped to disk by the `outputTooLong` mechanism, after which the agent
cannot actually *see* the image without a manual decode pipeline:

1. `grep` the dumped JSON for the marks/labels;
2. offset-`Read`s of the dump (one `Read` errored outright:
   "File content (431041 tokens) exceeds maximum allowed tokens");
3. hand-write PowerShell to base64-decode the blob to a `.png`.

The decode itself was non-obvious because the persisted payload wraps the image
under the MCP envelope: the base64 lives at `j.structuredContent.screenshot.base64`,
**not** top level. The agent's first two PowerShell attempts failed
("wrote 0 bytes; marks_drawn=" — it assumed `j.screenshot.base64`) and only the
third worked after discovering "top keys: content, structuredContent, isError".

Net: a single visual-review intent (observe chrome + eyeball a marked screenshot)
cost a grep, multiple Reads, and three PowerShell attempts on top of the RPC —
pure delivery friction, since the image bytes were fully recoverable the whole time.

## What it should do
Give `drive.observe` a file-delivery mode for the screenshot, mirroring how the
other screenshot verbs already return a saved `path`:

- e.g. `screenshot:"file"` (or a `screenshot_path` field / a `screenshot_mode`
  param) writes the Set-of-Mark PNG to disk and returns `{path, width, height,
  marks_drawn}` instead of the inline base64. The element list then stays small
  and the agent can `Read` the image directly.

The `outputTooLong` machinery already writes big payloads to disk — this just
routes the PNG through that path deliberately instead of forcing the caller to
reconstruct it from a dumped envelope. Weaker alternative: at minimum, document
on the wiki that with `screenshot:true` the base64 lands under
`structuredContent.screenshot.base64` and give a copy-paste decode snippet, so the
extraction isn't trial-and-error.

## Evidence
- `drive.observe {surface:editor_chrome, interactables_only:true, screenshot:true}`
  returned `outputTooLong`: "Response exceeds display limit (1094411 chars,
  threshold 10000); full payload written to ...json" (transcript line 174).
- The ~1MB is dominated by the inline base64 SoM PNG (decoded 80396 bytes).
- Multi-step base64 extraction incl. the wrote-0-bytes failure at transcript
  lines 174–346; a `Read` hit the token cap; the base64 was at
  `structuredContent.screenshot.base64`, not top level.
- Related low-friction confirmation of the same verbose-response family: this
  task's `drive.click` normal diff response (20434 chars) also tripped the
  10000-char `outputTooLong` threshold and dumped to disk (line 423), though that
  one was resolved in one `Read`.
- Call-trace source: `record.efficiency` inefficiency #2 (pattern=workaround),
  transcript
  `.../subagents/workflows/wf_60865404-cf4/agent-ae05318e07add50c7.jsonl`.

severity rationale: impact=response-spill that forces a decode workaround beyond
a plain Read (a grep + several Reads + 3 PowerShell attempts to view one image) ×
reach=screenshot:true is a drive.observe visual-review sub-path, not every-session
-> Low

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
