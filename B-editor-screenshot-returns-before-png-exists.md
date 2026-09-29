---
id: B-editor-screenshot-returns-before-png-exists
title: "editor.screenshot (PIE game viewport) returns success with a path before the PNG exists on disk"
status: OPEN
severity: Medium
category: bug
tags: [editor, screenshot, pie, async, file-write, scripting]
encounters: 1
lastSeen: 2026-09-29T09:57:00Z
---

# editor.screenshot returns before its PNG is on disk

## Symptom

On the PIE game-viewport path (`captureSource:"gameViewport"`, `captureMode:"nativeBackBuffer"`),
`editor.screenshot` answers with a success payload and a `path`, but the file at that path does
not exist yet when the response arrives. It appears a moment later.

The wiki says the capture "Runs as a synchronously-completed tracked job" and that "the returned
ticket is already terminal", which reads as "the file is written when the call returns".

## Repro (UE 5.8, Linux, PDS map editor in PIE, `/sdb-disk/src/unreal/unreal-fpv`)

A Python client over the MCP HTTP gateway (`POST http://127.0.0.1:<gateway-port>/mcp`,
`tools/call` -> `call` -> `editor.screenshot {filename:"survey_<n>"}`) followed each response
immediately with `shutil.copy(Saved/Screenshots/survey_<n>.png, ...)`. All 9 of 9 copies failed
with `FileNotFoundError`. `ls -t Saved/Screenshots` right afterwards showed all 9 files present.
Polling for the file (up to 10 s, then +0.2 s) made every later capture reliable (~150 shots).

A human-paced MCP client never sees this because the next tool call comes seconds later.

## Expected

Either the response is sent only after the PNG is fully written, or the docs and the response
say the write is asynchronous and give a completion signal (for example a `written:false` field
plus a job id to poll).

## Workaround

Poll for the file to exist with non-zero size before reading it.

## History

- `#1-initial-report` `OPEN` reporter - Found while scripting before/after captures for the PDS map-editor selection outline prototype (lighting sweep in L_PDS_ChemicalPlant). Evidence is above. The handler source was not inspected, so the root cause is unknown: an async PNG encode/write after the job is marked terminal is the likely candidate.
