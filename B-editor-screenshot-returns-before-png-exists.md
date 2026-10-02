---
id: B-editor-screenshot-returns-before-png-exists
title: "editor.screenshot (PIE game viewport) returns success with a path before the PNG exists on disk"
status: IN-REVIEW
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
- `#2-reply-after-capture` `IN-REVIEW` developer - Root cause: `FHandlerContext::StartJob` sent the non-streaming `status:"running"` envelope (with `requested_path`) to the transport BEFORE invoking the bind delegate, so the I/O thread could flush the reply while the game thread was still capturing and writing the PNG. Fix: new opt-in `FJobBindArgs::bCompletesInBind` (HandlerContext.h/.cpp) - StartJob then replies AFTER the delegate with the terminal outcome: success = started payload + job result (`path`, `width`, ...) + `ticket_id` + `status:"completed"`; failure = error reply with the job's error code and message, data carries `ticket_id` + `status:"failed"`. Streaming requests are unchanged; verbs that do not opt in are unchanged. `editor.screenshot` (ViewportHandler.cpp) opts in, so a success reply now means the PNG is fully written. Wiki: editor.md, system.md, mcp-transport.md, visual-review.md. Tests: `PinWright.infra.start_job.CompletesInBindRepliesWithResult`, `PinWright.infra.start_job.CompletesInBindFailureIsAnError` (both fail if StartJob replies before the delegate), and the reply assertions added to `PinWright.editor.screenshot.LevelViewportFallback` / `.GameViewportCompletesSynchronously` (reply not `running`, reply `path` exists on disk).
- `#3-linux-verification` `IN-REVIEW` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Passed in w23-final: `PinWright.infra.start_job.CompletesInBindRepliesWithResult` and `PinWright.infra.start_job.CompletesInBindFailureIsAnError` (StartJob replies after the bind delegate) and `PinWright.editor.screenshot.LevelViewportFallback` (reply not `running`, reply `path` exists on disk). Not demonstrated: the reported path. `PinWright.editor.screenshot.GameViewportCompletesSynchronously`, the only test of `captureSource:gameViewport` / `nativeBackBuffer`, skipped `no-game-viewport` in w23-final and is not in the X runs, so whether the PIE back-buffer capture writes its PNG inside the delegate is unmeasured. Needs: a run with a bound PIE game viewport where that test passes unskipped, or the #1 repro (copy the file immediately after each PIE `editor.screenshot`). The newly filed `B-startjob-inline-verbs-reply-running-before-work` covers other StartJob verbs.
