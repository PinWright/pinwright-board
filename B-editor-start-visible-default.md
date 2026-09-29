---
id: B-editor-start-visible-default
title: "editor_start / editor_restart silently default visible to true, so an agent that omits it gets a windowed editor it never asked for"
status: IN-REVIEW
severity: Medium
category: bug
tags: [proxy, editor-start, editor-restart, lifecycle, visible, defaults, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T00:00:00Z
---

# editor_start / editor_restart default `visible` to true

`Content/Python/mcp_proxy.py` `_editor_start` read `visible = args.get("visible", True)` and
`editor_restart` forwarded `visible` only when present, so a call without it launched a normal
windowed editor. An agent that wanted a headless run and forgot the parameter got a visible editor
(uncapped, dialogs enabled) instead of an error. User requirement: the launch mode must be demanded
explicitly, with no default.

**Fix:** `visible` is a required boolean on both tools (schema `required`, descriptions without
"(default)"); a missing value is refused with `MISSING_REQUIRED_PARAM` and a non-boolean with
`INVALID_VISIBLE`, both naming what `visible:true` and `visible:false` launch, before the live-editor
guard runs and before `editor_restart` stops anything. `editor_restart` forwards the value verbatim.

## History
- `#1-initial-report` `OPEN` reporter — `visible` defaulted to `true` in `_editor_start` and was optional on `editor_restart`; an omitted value launched a visible editor.
- `#2-visible-required` `IN-REVIEW` developer — `Content/Python/mcp_proxy.py`: `visible` (and `reason`) added to both schemas' `required`; `Proxy._launch_intent` refuses missing `visible` with `MISSING_REQUIRED_PARAM` and a non-boolean with `INVALID_VISIBLE`, naming what each value launches, before the guard and before `editor_restart` stops anything; restart forwards `visible` verbatim. Tests in `tests/test_mcp_proxy_editor_start.py` cover refusal for both tools without spawning and pin the visible:true / visible:false command lines. `unittest discover tests`: 320 OK (1 skipped).
- `#3-mode-enum-replaces-visible` `IN-REVIEW` developer — User decision: the `visible` boolean is replaced by a required enum `mode` (`visible` | `offscreen` | `headless`, no default, no `visible` alias) on editor_start, editor_restart, editor_run_tests and the `pinwright_launch.py --mode` CLI. Missing -> `MISSING_REQUIRED_PARAM`; any other value -> `INVALID_MODE` (replaces `INVALID_VISIBLE`), message lists all three. `headless` = `-NullRHI` plus the offscreen flags (`-RenderOffScreen` kept: it selects the null platform application and Linux SDL dummy driver, so no window and no display). Python suite 325 OK (1 skipped).
