---
id: E-session-split-screen-type-param
title: "session.set_split_screen_type rejects the obvious 'type' param (requires verbose 'splitScreenType')"
status: OPEN
severity: Low
category: ergonomic
tags: [session, param-alias]
encounters: 1
lastSeen: 2026-06-17T02:15:04Z
---

# session.set_split_screen_type rejects the obvious `type` param

`session.set_split_screen_type` takes a single argument — the layout enum
(`None`, `TwoPlayer_Horizontal`, …) — but the required param is named
`splitScreenType`, which redundantly repeats the method name. The natural
shorthand a caller reaches for is `type`, and that is rejected with
`UNKNOWN_PARAMS`. Same friction class as `E-material-editor-param-name-drift`,
`E-widget-remove-widget-param-name`, and `E-blueprint-param-name-path-vs-assetpath`
(all DONE): the obvious param name guesses wrong and eats a round-trip.

The param spec lives at `SessionsHandler.cpp:196`
(`RPC_PARAM_OPT("splitScreenType", …)`), read at `:199`. Its sibling
`session.configure_split_screen` also takes the same value under the same
verbose name (`SessionsHandler.cpp:143`), so a single `type` alias applied to
both keeps the pair consistent. When a method takes exactly one descriptive
argument, the un-prefixed `type` is the discoverable name; forcing the
method-name-echoing `splitScreenType` is pure memorization tax.

**Repro:** `session.set_split_screen_type {type:"TwoPlayer_Horizontal"}` →
`[UNKNOWN_PARAMS] Unknown parameter(s) for 'session.set_split_screen_type':
[type]. Valid parameters: [splitScreenType].` Retry with
`{splitScreenType:"TwoPlayer_Horizontal"}` succeeds. (The error hint names the
valid param, so the cost is one wasted round-trip, not a hard block — hence Low.)

**Fix:** Add `type` (and, for parity with the snake_case alias convention,
`split_screen_type`) as `RPC_PARAM_OPT` aliases on both
`session.set_split_screen_type` and `session.configure_split_screen`, reading
via `GetStringFirstOf({splitScreenType, type, split_screen_type})`. Keep
`splitScreenType` canonical so existing callers keep working. Aliases only — no
payload rewriting — per the dispatcher `FParamSpec` alias machinery established
in `E-blueprint-param-name-path-vs-assetpath #4`.

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced by a session split-screen
  teardown task (focus `session.remove_local_player`, outcome agent_fail). In a
  9-call run, the agent's first `session.set_split_screen_type` call used
  `type=TwoPlayer_Horizontal` and got `[UNKNOWN_PARAMS] … Valid parameters:
  [splitScreenType]`; it corrected on the next call with
  `splitScreenType=TwoPlayer_Horizontal` (1 wasted call). Friction note:
  "one wrong param name on set_split_screen_type (type vs splitScreenType, fixed
  on retry via the error hint)." Process angle only — the task's hard blocker
  (`[NO_GAME_INSTANCE]` on add/remove_local_player, no live PIE on this host) is
  an outcome matter owned by the per-finding judge, not this ticket. Confirmed
  against source: `SessionsHandler.cpp:196` (`splitScreenType` req on
  set_split_screen_type) and `:143` (same name on configure_split_screen). No
  existing board ticket on session/split-screen param naming (ripgrep + qmd,
  OPEN and closed).
