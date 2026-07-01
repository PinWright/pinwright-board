---
id: E-session-info-split-screen-type-not-layout
title: "session.get_sessions_info reports splitScreenType as 'Active'/'None' (a player-count flag), not the layout enum the setters take — readback can't confirm the layout"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [session, readback, split-screen, field-name-collision]
---

# `session.get_sessions_info` reports `splitScreenType` as `Active`/`None`, not the configured layout

`session.set_split_screen_type` and `session.configure_split_screen` both take
and echo a `splitScreenType` field whose value is the layout **enum**
(`None`, `TwoPlayer_Horizontal`, `TwoPlayer_Vertical`, `ThreePlayer_FavorTop`,
`ThreePlayer_FavorBottom`, `FourPlayer_Grid`). The obvious confirm-step — call
`session.get_sessions_info` and read back `splitScreenType` — returns a field of
the **same name** but a different value domain: it is hardcoded to
`Active` / `None` based purely on player count, never the configured layout. So
a caller doing a natural round-trip readback of `splitScreenType` gets a string
that looks like it should be the layout enum, isn't, and silently cannot confirm
which layout is in effect.

The collision is in the source, not just the response:

- Getter — `SessionsHandler.cpp:746`:
  `SessionsInfo->SetStringField(TEXT("splitScreenType"), LocalPlayerCount > 1 ? TEXT("Active") : TEXT("None"));`
  The value is derived only from `GetLocalPlayerCount()`; the configured layout
  enum is never stored anywhere and never read back. (`splitScreenEnabled` on
  the line above, `:745`, is likewise just `LocalPlayerCount > 1` — these two
  fields are runtime player-count flags wearing split-screen names.)
- Setters echo the enum — `SessionsHandler.cpp:214`
  (`session.set_split_screen_type`) and `:180`
  (`session.configure_split_screen`) both
  `SetStringField(TEXT("splitScreenType"), SplitScreenType)` where
  `SplitScreenType` is the literal enum the caller passed.

Because the same field name carries the layout enum on the write path but an
on/off runtime string on the read path, the documented "re-check the session
status to confirm the split-screen layout took effect" flow is structurally
incapable of confirming the layout. Layout confirmation has to fall back to the
set/configure call echoes plus player counts — the getter contributes nothing.

Note this is **distinct** from `E-session-split-screen-type-param` (OPEN), which
is about the *input* param name (`splitScreenType` vs the shorthand `type`) on
the setter. This ticket is about the *output* field of the getter reporting the
wrong value domain under that same name.

**Repro** (live, PIE running with split-screen configured):
1. `session.set_split_screen_type {splitScreenType:"ThreePlayer_FavorTop"}` →
   `{"splitScreenType":"ThreePlayer_FavorTop"}` (echoes the layout enum).
2. `session.get_sessions_info {}` →
   `{"sessionsInfo":{... "splitScreenEnabled":true, "splitScreenType":"Active", ...}}`
   — the readback of the **same field name** reports `"Active"`, not
   `"ThreePlayer_FavorTop"`. Setting `TwoPlayer_Horizontal` and reading back
   again likewise yields `"splitScreenType":"Active"`. The getter never echoes
   any layout enum; it only ever reports `Active`/`None`.

**Workaround:** confirm the configured layout from the
`session.set_split_screen_type` / `session.configure_split_screen` response
echo (which returns the enum verbatim); treat `get_sessions_info`'s
`splitScreenType` only as an enabled/disabled flag, not a layout name.

**Fix:** lowest-cost is to stop reusing the `splitScreenType` field name for the
on/off flag in the getter — rename the `get_sessions_info` field to something
like `splitScreenState` (values `Active`/`None`), leaving `splitScreenEnabled`
as the boolean. Better: have the setters persist the configured layout enum into
session/plugin state and have `get_sessions_info` echo that stored enum back
under `splitScreenType`, so the write→read round-trip closes. Either removes the
same-name/different-meaning trap.

## History
- `#1-initial-repro` `OPEN` reporter — Surfaced by a couch co-op split-screen
  validation task (focus `session.get_sessions_info`, outcome ergo). The agent
  ran the full flow successfully (PIE start, add 2 local players 1→3, enable
  split-screen + `ThreePlayer_FavorTop`, remove highest-index player, reconfigure
  `TwoPlayer_Horizontal`) and noted in friction that `get_sessions_info` reports
  `splitScreenType` only as `Active`/`None` runtime state, not the literal layout
  enum, so layout confirmation relied on the set/configure echoes plus player
  counts. Replay-confirmed live: after
  `set_split_screen_type {splitScreenType:"ThreePlayer_FavorTop"}` (echoed
  verbatim), `get_sessions_info` returned `"splitScreenType":"Active"`. Grounded
  in source: getter hardcodes `LocalPlayerCount > 1 ? "Active" : "None"`
  (`SessionsHandler.cpp:746`); setters echo the enum (`:214`, `:180`). Distinct
  from `E-session-split-screen-type-param` (input param name on the setter) —
  this is the getter's output field value domain. No existing board ticket on the
  `get_sessions_info` split-screen readback (ripgrep, OPEN + closed).
- `#2-additional-fourplayer-grid` `OPEN` reporter — Additional evidence: a 4-player
  couch co-op setup task (REALISM mode) asked for the `FourPlayer_Grid` layout and a
  final `get_sessions_info` readback "to confirm split-screen and the local players
  are all configured the way I asked." Replay-confirmed live (PIE running, 4 local
  players): `session.set_split_screen_type {splitScreenType:"FourPlayer_Grid"}` echoed
  `{"splitScreenType":"FourPlayer_Grid"}`, then `session.get_sessions_info {}` returned
  `{"sessionsInfo":{"localPlayerCount":4,"inPlaySession":true,"currentSessionName":"None","isLANMatch":false,"maxPlayers":0,"currentPlayers":4,"splitScreenEnabled":true,"splitScreenType":"Active","voiceChatEnabled":false,"isHosting":false,"connectedServerAddress":"","activeVoiceChannels":[]}}`
  — the readback reports `"splitScreenType":"Active"`, not `"FourPlayer_Grid"`, so the
  layout name still cannot be confirmed via the getter. Same failure mode as #1 with a
  third distinct layout enum (after `ThreePlayer_FavorTop` and `TwoPlayer_Horizontal`),
  confirming the getter never echoes ANY layout value regardless of which one was set.
  Source unchanged (`SessionsHandler.cpp:767`, `LocalPlayerCount > 1 ? "Active" : "None"`).
  The same task's other friction (hardcoded `voiceChatEnabled:false`, the PIE
  prerequisite on `add_local_player`, and the `enable_voice_chat` host-plugin error)
  is covered by `E-session-info-readback-hardcoded` and
  `E-session-wiki-pie-prerequisite-undocumented`. Grouped here rather than a new ticket
  (duplicate of OPEN).
- `#3-retriage` `OPEN` triage — Low→Medium: splitScreenType reads back Active/None not the configured layout enum under the same field name, silent-wrong readback on a niche couch-coop session path.
- `#4-fix` `IN-REVIEW` developer — Closed the split-screen layout write→read round-trip the ticket's primary "Better" Fix names, extending the `FConfiguredSessionSettings` infrastructure the sibling `E-session-info-readback-hardcoded` #4-fix already landed (now present in source — no new state machinery). Added a `SplitScreenLayout` field (+ `bSplitScreenConfigured` honesty guard, independent of the identity setter's `bConfigured`) to `FConfiguredSessionSettings` on `FPluginState`. Both split-screen setters now persist the enum: `session.set_split_screen_type` stores the validated layout (after the enum-list check), and `session.configure_split_screen` stores the requested layout (or "None" when disabled). `session.get_sessions_info` now reads `splitScreenType` back from that stored enum, falling back to the prior live player-count flag ("Active"/"None") only while `bSplitScreenConfigured` is still false, so it never emits an empty/stale enum and never lies on a fresh session. `splitScreenEnabled` is untouched (still the live `LocalPlayerCount > 1` bool). Files: `Source/PinWright/Private/State/PluginState.h` (new `SplitScreenLayout`/`bSplitScreenConfigured` fields), `Source/PinWright/Private/Handlers/System/SessionsHandler.cpp` (two setters persist, getter reads back). `FConfiguredSessionSettings{}` reset in `ResetForTesting` already zeroes the new fields. Regression test `PinWright.session.get_sessions_info.ReadsBackSplitScreenLayout` in `Source/PinWright/Private/Tests/EditorOps/TestSystemHandlers.cpp` sets `ThreePlayer_FavorTop` then `FourPlayer_Grid` via `set_split_screen_type` and `TwoPlayer_Vertical` via `configure_split_screen`, asserting `get_sessions_info` reads each layout back verbatim; it fails on pre-fix code where the getter only ever returned "Active"/"None". Note: the board-historian lens voted defer-until-sibling-lands, but the sibling's infrastructure is already merged into current source (the plugin clone synced to origin before this run), so the named gate is satisfied — the disposition collapses to GO and the fix is a clean one-field extension with no collision.
