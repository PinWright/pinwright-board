---
id: E-session-wiki-pie-prerequisite-undocumented
title: "session wiki overlay doesn't say which session.* methods require live PIE — callers eat a [NO_GAME_INSTANCE] on the first add/remove_local_player"
status: OPEN
severity: Low
category: ergonomic
tags: [session, docs, pie-prerequisite, discoverability, voice-chat-plugin]
encounters: 5
lastSeen: 2026-07-01T16:42:43.2664492+03:00
---

# `session` wiki overlay omits the live-PIE prerequisite for local-player / split-screen methods

Most `session.*` methods that touch the local-player roster or split-screen
config operate on the **active game instance**, which only exists while
Play-In-Editor is running. With PIE stopped, the internal
`SessionsHandler::GetGameInstance()` helper (`SessionsHandler.cpp:44`, returns
`GEditor->PlayWorld->GetGameInstance()`) is null and the handler aborts with
`[NO_GAME_INSTANCE] No active game instance. Start Play-In-Editor first.`
(`SessionsHandler.cpp:232` for `add_local_player`, `:268` for
`remove_local_player`). `session.get_sessions_info`, by contrast, works
pre-PIE (returns `count 0, inPlaySession:false`), so the two halves of the
namespace have different prerequisites with nothing in the docs to flag the
split.

The `session` wiki overlay (`docs/wiki-src/session.md`) is a single descriptive
line enumerating what the namespace covers (local player add/remove, split-screen
layout, travel, voice chat, …). It does **not** state that the roster/split-screen
methods require a live PIE world, nor that they fail with `[NO_GAME_INSTANCE]`
otherwise. So a caller following a natural "set up local multiplayer" intent
reaches for `add_local_player` first and discovers the prerequisite only by
eating a failed call.

This is purely a **discoverability/process** gap — the methods behave correctly
and the error text even names the remedy ("Start Play-In-Editor first"), so the
cost is one wasted round-trip per fresh session, not a hard block. Distinct from
the two existing session split-screen tickets, which are about the setter input
param name (`E-session-split-screen-type-param`) and the getter's output field
value domain (`E-session-info-split-screen-type-not-layout`) — neither documents
the PIE prerequisite.

**Evidence (recurring across tasks):**
- This task (focus `session.get_sessions_info`, outcome ergo, 13 calls): the
  agent's first `session.add_local_player {controllerId:-1}` returned
  `[NO_GAME_INSTANCE] No active game instance. Start Play-In-Editor first.`; it
  then ran `editor.play` and the rest of the flow (add 2 local players 1→3,
  split-screen + `ThreePlayer_FavorTop`, remove highest-index player,
  `TwoPlayer_Horizontal`) succeeded. Friction note: "add_local_player needs an
  active game instance, so the first call failed with [NO_GAME_INSTANCE] until I
  started PIE via editor.play — the get_sessions_info baseline (count 0,
  inPlaySession:false) made the cause obvious." (One wasted call; the baseline
  readback made the diagnosis quick, but the wiki could have pre-empted it.)
- A prior session split-screen teardown task (focus
  `session.remove_local_player`, outcome agent_fail) hit the same
  `[NO_GAME_INSTANCE]` on add/remove_local_player with no live PIE on that host —
  recorded in `E-session-split-screen-type-param #1`, which explicitly deferred
  the PIE blocker as an outcome matter and did **not** file the docs angle.
- A LAN co-op client-join task (focus `session.join_lan_server`, outcome
  tool_bug, 13 calls): the agent's first `session.add_local_player
  {controllerId:-1}` (call #7) returned `[NO_GAME_INSTANCE] No active game
  instance. Start Play-In-Editor first.`; it then ran `editor.play` (call #10)
  and the retry `session.add_local_player` (call #11) succeeded with
  `totalLocalPlayers=2`. Note the ordering cost here: the agent had already run
  `configure_lan_play` / `configure_session_interface` / `join_lan_server` /
  `configure_split_screen` / `set_split_screen_type` BEFORE reaching for
  `add_local_player`, so the PIE prerequisite surfaced mid-flow — one wasted call
  plus an out-of-order PIE start in the middle of a "configure the client" intent
  that a doc note would have front-loaded. Friction note: "add_local_player
  failed with NO_GAME_INSTANCE until I started PIE via editor.play." Third task
  recurrence of the same overlay gap.

A **sibling precondition gap in the same overlay**: the `session.md` line also
advertises "voice chat ... proximity attenuation" as a supported capability, but
`session.enable_voice_chat` hard-fails with
`[VOICE_CHAT_ERROR] IVoiceChat interface not available - no voice chat plugin
loaded` on any host that doesn't load a voice-chat plugin (e.g. this fuzz host).
The overlay flags neither the host-plugin dependency nor that the related setters
(`set_voice_attenuation`, push-to-talk, muting) configure attenuation params even
on a host where voice chat can never enable — so a caller following the same
natural "set up couch-coop voice" intent reaches for `enable_voice_chat`, eats
the error, and (as in this task) **retries the identical call expecting a
different result** after an unrelated `set_voice_attenuation` in between. Same
overlay-omits-the-precondition failure mode as the PIE gap, different
precondition (plugin-loaded vs PIE-running).

**Fix (downstream wiki process):** extend the `docs/wiki-src/session.md` overlay
to note (a) that the local-player roster and split-screen methods
(`add_local_player`, `remove_local_player`, `configure_split_screen`,
`set_split_screen_type`, travel) operate on the active game instance and require
PIE to be running, returning `[NO_GAME_INSTANCE]` otherwise, while
`get_sessions_info` is safe to call pre-PIE for a baseline; and (b) that
`enable_voice_chat` requires a voice-chat plugin to be loaded on the host and
returns `[VOICE_CHAT_ERROR] IVoiceChat interface not available` otherwise — a
host-capability gap that retries can't clear, so callers shouldn't re-attempt it.
This is a wiki edit, not a code change.

## History
- `#1-initial-audit` `OPEN` reporter — Process/struggle audit of a couch co-op
  split-screen validation task (focus `session.get_sessions_info`, outcome ergo,
  13 calls). The agent's first `session.add_local_player` failed with
  `[NO_GAME_INSTANCE] No active game instance. Start Play-In-Editor first.` before
  it started PIE via `editor.play`; rest of the flow was clean. The session wiki
  overlay (`docs/wiki-src/session.md`, one line) documents no PIE prerequisite, so
  the requirement is discovered only by the failed call. Grounded in source:
  `SessionsHandler::GetGameInstance()` returns `GEditor->PlayWorld->GetGameInstance()`
  (`SessionsHandler.cpp:44`); `add_local_player`/`remove_local_player` send
  `NO_GAME_INSTANCE` when it's null (`:232`, `:268`). Recurring: same blocker
  recorded earlier in `E-session-split-screen-type-param #1` (deferred as outcome,
  docs angle never filed). Distinct from the two existing session split-screen
  tickets (input param name; getter output domain). No existing board ticket on
  the session-namespace PIE-prerequisite docs gap (ripgrep across OPEN + closed;
  qmd unavailable).
- `#2-voice-chat-host-plugin` `OPEN` reporter — Aggregated a sibling precondition
  gap in the **same `docs/wiki-src/session.md` overlay** from a couch co-op
  voice-setup task (focus `session.get_sessions_info`, outcome ergo, 17 calls).
  The overlay advertises "voice chat / proximity attenuation" as supported, but
  `session.enable_voice_chat {voiceEnabled:true}` hard-failed with
  `[VOICE_CHAT_ERROR] IVoiceChat interface not available - no voice chat plugin
  loaded`; the agent then **retried the identical call** (call #11 → #13, after an
  unrelated `set_voice_attenuation {radius:1500, falloff:1.5}`) and got the same
  error — a wasted trial-and-error retry on a host-capability gap the overlay
  never flags. Friction note: "enable_voice_chat needs a voice-chat plugin this
  host doesn't load." Same overlay-omits-the-precondition failure mode as the PIE
  gap (one downstream wiki edit covers both), different precondition
  (plugin-loaded vs PIE-running). Distinct from the judge's outcome ticket
  `E-session-info-readback-hardcoded` (which owns the hardcoded `voiceChatEnabled`
  getter / unverifiable readback) — this is the docs/discoverability +
  wasted-retry process half. No separate board ticket on `enable_voice_chat`
  discoverability (ripgrep across OPEN + closed; qmd unavailable).
- `#3-pie-prereq-third-recurrence` `OPEN` reporter — Process/struggle audit of a
  LAN co-op client-join task (focus `session.join_lan_server`, outcome tool_bug,
  13 calls). Third task recurrence of this same overlay gap: the agent's first
  `session.add_local_player {controllerId:-1}` (call #7) returned
  `[NO_GAME_INSTANCE] No active game instance. Start Play-In-Editor first.`; it
  then ran `editor.play` (call #10) and the retry succeeded
  (`totalLocalPlayers=2`, call #11). Aggravating detail vs the prior two
  instances: the agent had already run the four pre-PIE-safe config calls
  (`configure_lan_play`, `configure_session_interface`, `join_lan_server`,
  `configure_split_screen` + `set_split_screen_type`) before hitting the
  prerequisite on `add_local_player`, so PIE had to be started out-of-order in the
  middle of the "configure the client" flow — exactly the front-loading the
  overlay note would enable. Friction note: "add_local_player failed with
  NO_GAME_INSTANCE until I started PIE via editor.play." No new ticket; appended
  as cross-task evidence. The task's other friction (the get_sessions_info
  readback being a hardcoded dead end — isLANMatch/connectedServerAddress/
  splitScreenType never round-trip) is the OUTCOME, owned by the judge's
  `B-join-lan-server-malformed-url` and the existing readback tickets
  (`E-session-info-readback-hardcoded`, `E-session-info-split-screen-type-not-layout`),
  not a distinct process angle.
- `#4-pie-prereq-redo-and-voice-retry` `OPEN` reporter — Process/struggle audit of
  a 4-player couch co-op split-screen + voice setup task (focus
  `session.get_sessions_info`, outcome ergo, 26 calls). **Fourth recurrence of the
  PIE-prerequisite gap, with a new aggravating wrinkle — wasted REDO work, not just
  one wasted call:** because `configure_split_screen` / `set_split_screen_type`
  silently *succeed* pre-PIE (they `SaveSettings()` + echo and never call
  `GetGameInstance()` — `SessionsHandler.cpp:154`,`:208`, no `NO_GAME_INSTANCE`
  guard, unlike `add_local_player`/`remove_local_player` at `:246`/`:282`), the
  agent ran `configure_split_screen` (call #11) and `set_split_screen_type`
  (`FourPlayer_Grid`, call #12) pre-PIE and both returned ok — lulling it into
  thinking setup was progressing. Only the next call `add_local_player
  {controllerId:1}` (call #13) hard-failed with `[NO_GAME_INSTANCE] No active game
  instance. Start Play-In-Editor first.` The agent then ran `editor.play` (call #15)
  and had to **re-run `configure_split_screen` (call #17) and `set_split_screen_type`
  (call #18) all over again** post-PIE before the three `add_local_player` calls
  (1→2→3→4, calls #19-21) could succeed. So the overlay gap cost two redone setter
  calls here, because the inconsistency between the silently-pre-PIE-OK split-screen
  setters and the hard-failing roster methods made the prerequisite invisible until
  mid-flow. A doc note stating "the split-screen setters appear to succeed pre-PIE
  but only the SaveSettings echo applies; start PIE first so the whole config lands
  in one pass" would front-load this. **Also a 2nd recurrence of `#2-voice-chat-host-plugin`
  (identical-call retry):** `session.enable_voice_chat {voiceEnabled:true}` failed
  with `[VOICE_CHAT_ERROR] IVoiceChat interface not available - no voice chat plugin
  loaded` (call #22), and after an unrelated `configure_voice_settings` +
  `configure_push_to_talk` the agent **retried the identical call** (call #25) and
  got the same error — same wasted trial-and-error on a host-capability gap the
  overlay never flags. Friction note: "add_local_player … require a live PIE game
  instance but their wiki pages never say so … forcing an editor.play detour" and
  "enable_voice_chat returns a hard error because this project ships no voice-chat
  provider plugin." No new ticket; appended as cross-task evidence. The task's other
  friction (get_sessions_info hardcodes `splitScreenType="Active"` and
  `voiceChatEnabled=false`) is the OUTCOME, owned by the judge's filed
  `E-session-info-split-screen-type-not-layout` (now carrying the `FourPlayer_Grid`
  evidence as its `#2`) and `E-session-info-readback-hardcoded`, not a process angle.
- `#5-pie-liveness-and-voice-sourcedive` `OPEN` reporter — Process/struggle audit of a
  2-player couch co-op split-screen + proximity-voice setup task (focus
  `session.get_sessions_info`, outcome ergo, 14 real calls). **PIE half — pure
  liveness (5th recurrence), cleanest form:** the agent reached for
  `session.add_local_player {controllerId:-1}` FIRST on a freshly-loaded editor (before
  any split-screen setters), got `[NO_GAME_INSTANCE] No active game instance. Start
  Play-In-Editor first.`, ran `editor.play`, retried → `totalLocalPlayers=2`. One
  wasted call, no out-of-order redo (unlike `#3`/`#4`); the overlay gap is unchanged,
  still observed. **Voice half — 3rd recurrence, NEW wrinkle (source dive instead of
  retry):** `session.enable_voice_chat {voiceEnabled:true}` hard-failed with
  `[VOICE_CHAT_ERROR] IVoiceChat interface not available - no voice chat plugin loaded`,
  but unlike `#2`/`#4` the agent did NOT retry it. Instead, because its four sibling
  voice methods in the SAME batch all returned success (`configure_voice_settings
  {volume:0.8,noiseSuppression:true,echoCancel:true}`, `configure_push_to_talk
  {enabled:true,key:V}`, `set_voice_attenuation {radius:2000,falloff:2}`,
  `set_voice_channel {name:CoopPair,type:Proximity}`), the false "it's wired up"
  impression forced the agent to Grep+Read plugin C++ (`SessionsHandler.cpp`) to confirm
  the enable failure was STRUCTURAL (host-plugin-dependent) and not its own error — a
  source-dive cost the overlay note would eliminate outright. Friction note: "the four
  voice sub-config methods all return success and store state, giving a false 'it's wired
  up' impression; no wiki page warns a voice provider plugin is required. Had to read
  plugin C++ (SessionsHandler.cpp) to confirm it was structural, not my error." Same
  overlay-omits-the-precondition gap this ticket owns; the fix (extend
  `docs/wiki-src/session.md` to flag the IVoiceChat plugin dependency AND that the voice
  sub-setters only stage settings on a host where voice can never enable) is unchanged.
  The readback-hardcoded OUTCOME (`voiceChatEnabled` live-only, `activeVoiceChannels`
  hardcoded `[]`) is owned by the judge's `E-session-info-readback-hardcoded`
  (this exact task appended there as its `#5-additional-voice-channel-readback`, which
  explicitly delegates the false-"wired up" discoverability process half here); not a
  separate ticket. No new ticket; appended as cross-task evidence.
