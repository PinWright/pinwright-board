---
id: E-session-info-readback-hardcoded
title: "session.get_sessions_info hardcodes currentSessionName='None', maxPlayers=0, voiceChatEnabled=false — readback can't confirm the configured session name, max players, or voice state"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [session, readback, hardcoded, no-op-setter]
encounters: 3
lastSeen: 2026-07-02T14:47:53.4842628+03:00
---

# `session.get_sessions_info` returns hardcoded `currentSessionName`/`maxPlayers`/`voiceChatEnabled`, never the configured values

`session.get_sessions_info` is the natural "confirm the session is set up the way
I asked" readback. But several of its fields are **hardcoded literals**, not the
values a caller just configured, so the round-trip can never confirm them:

- `currentSessionName` — always `"None"` (`SessionsHandler.cpp:741`), regardless
  of the `sessionName` passed to `session.configure_local_session_settings`.
- `maxPlayers` — always `0` (`SessionsHandler.cpp:743`), regardless of the
  `maxPlayers` configured.
- `voiceChatEnabled` — always `false` (`SessionsHandler.cpp:747`), regardless of
  `session.enable_voice_chat`.
- `isLANMatch` / `isHosting` / `connectedServerAddress` / `activeVoiceChannels`
  are likewise fixed (`false` / `false` / `""` / `[]`, `:742`,`:748`-`:752`).

Only `localPlayerCount` / `currentPlayers` / `inPlaySession` /
`splitScreenEnabled` are live (derived from the PIE game instance's local-player
roster). Everything that describes the *configured session* is a placeholder.

Root cause: the setter that takes these values,
`session.configure_local_session_settings` (`SessionsHandler.cpp:77`-`110`), is a
**pure echo** — it reads `sessionName`/`maxPlayers`/etc., echoes them straight
back in its own response, and stores them nowhere. There is no persistent
session-settings state for the getter to read, so `get_sessions_info` returns
fixed defaults. The write call "succeeds" and the read call "succeeds", but the
two cannot agree because nothing connects them.

This is the **session-identity** half of the same getter problem tracked for the
split-screen layout in `E-session-info-split-screen-type-not-layout` (OPEN):
there, `splitScreenType` reads back `Active`/`None` instead of the configured
layout enum. This ticket covers the distinct `currentSessionName` / `maxPlayers`
/ `voiceChatEnabled` fields, which that ticket does not mention. Same handler,
same "getter never echoes configured state" failure mode, different field set.

**Repro** (live, PIE running, 3 local players):
1. `session.configure_local_session_settings {sessionName:"CouchCoop", maxPlayers:4}`
   → `{"sessionName":"CouchCoop","maxPlayers":4, ...}` (echoes the config back).
2. `session.get_sessions_info {}` →
   `{"sessionsInfo":{"localPlayerCount":3,"inPlaySession":true,"currentSessionName":"None","isLANMatch":false,"maxPlayers":0,"currentPlayers":3,"splitScreenEnabled":true,"splitScreenType":"Active","voiceChatEnabled":false,"isHosting":false,"connectedServerAddress":"","activeVoiceChannels":[]}}`
   — `currentSessionName` is `"None"` (not `"CouchCoop"`), `maxPlayers` is `0`
   (not `4`), `voiceChatEnabled` is `false`. The configured session name and max
   players never appear in the readback.

**Workaround:** treat the `session.configure_local_session_settings` response
echo as the only confirmation that the requested name/maxPlayers were accepted;
do not expect `get_sessions_info` to reflect them. Use `get_sessions_info` only
for the live player-count / PIE fields, which are accurate.

**Fix:** have `session.configure_local_session_settings` (and
`session.enable_voice_chat`) persist the configured values into session/plugin
state, and have `get_sessions_info` read that stored state back under
`currentSessionName` / `maxPlayers` / `voiceChatEnabled`, so the write→read
round-trip closes. If persisting real session state is out of scope, the
honest alternative is to stop emitting these fixed fields (or document them as
"not currently tracked") so callers aren't misled into reading them as confirmation.

## History
- `#1-initial-repro` `OPEN` reporter — Surfaced by a couch co-op split-screen
  validation task (REALISM mode; the task asked to confirm "CouchCoop /
  maxPlayers 4 / voice / ThreePlayer_FavorTop" via a final `get_sessions_info`
  readback). The agent ran the flow successfully but noted the readback could
  not confirm the configured session name, max players, or voice state.
  Replay-confirmed live: after
  `configure_local_session_settings {sessionName:"CouchCoop", maxPlayers:4}`
  (echoed verbatim), `get_sessions_info` returned `currentSessionName:"None"`,
  `maxPlayers:0`, `voiceChatEnabled:false`. Grounded in source: getter hardcodes
  these fields (`SessionsHandler.cpp:741`,`:743`,`:747`); the setter
  `configure_local_session_settings` is a pure echo storing nothing
  (`:77`-`110`). Distinct from `E-session-info-split-screen-type-not-layout`
  (same getter, the `splitScreenType` field) — this is the
  `currentSessionName`/`maxPlayers`/`voiceChatEnabled` field set, which that
  ticket does not cover. No existing board ticket on the session-identity
  readback fields (ripgrep across OPEN + closed; qmd unavailable).
- `#2-additional-couch-coop-voice` `OPEN` reporter — Additional evidence: a
  second couch co-op task (REALISM mode) asked to set up 2-player split-screen
  with proximity voice (`TwoPlayer_Horizontal`, voice chat, mic volume 0.5,
  noise suppression + echo cancellation, attenuation radius 1500 / falloff 1.5,
  push-to-talk V) and then "read back the session info so I can confirm the
  two-player split-screen layout and the voice settings are actually in effect"
  — exactly the round-trip this getter cannot satisfy. Replay-confirmed live
  (PIE running, 2 local players): `session.get_sessions_info {}` →
  `{"sessionsInfo":{"localPlayerCount":2,"inPlaySession":true,"currentSessionName":"None","isLANMatch":false,"maxPlayers":0,"currentPlayers":2,"splitScreenEnabled":true,"splitScreenType":"Active","voiceChatEnabled":false,"isHosting":false,"connectedServerAddress":"","activeVoiceChannels":[]}}`
  — `voiceChatEnabled` is `false` even after the task configured voice, and the
  configured voice settings / attenuation / push-to-talk never appear in the
  readback at all (no `voiceSettings`/`pushToTalk`/`attenuation` fields). The
  `splitScreenType:"Active"` half is the sibling ticket
  `E-session-info-split-screen-type-not-layout`. Note in this run
  `session.enable_voice_chat` itself hard-errored cleanly
  (`[VOICE_CHAT_ERROR] IVoiceChat interface not available - no voice chat plugin
  loaded`, correct env-dependent error, not a bug), so `voiceChatEnabled:false`
  is arguably accurate here — but the echo setters that DID succeed
  (`set_voice_attenuation {attenuationRadius:1500,attenuationFalloff:1.5}` and
  `configure_push_to_talk {pushToTalkEnabled:true,pushToTalkKey:"V"}` both echo
  their input verbatim, storing nothing) are likewise invisible in the readback,
  reinforcing the core "setters echo, getter hardcodes, round-trip can't close"
  failure mode. Source re-confirmed: getter still hardcodes these fields
  (`SessionsHandler.cpp:762`-`770`; `voiceChatEnabled` literal `false` at `:768`,
  `splitScreenType` `LocalPlayerCount>1?"Active":"None"` at `:767`). Grouped
  here rather than filing a new ticket (duplicate of OPEN).
- `#3-retriage` `OPEN` triage — Low→Medium: getter returns hardcoded currentSessionName/maxPlayers/voiceChatEnabled while setter only echoes, silent-wrong readback that can never confirm config on a niche couch-coop session path.
- `#4-fix` `IN-REVIEW` developer — Closed the write→read round-trip the ticket's primary Fix names, using the existing `FPluginState` singleton (no new state infrastructure). `session.configure_local_session_settings` now persists the configured `sessionName`/`maxPlayers`/`bIsLANMatch` into a new `FConfiguredSessionSettings` field on `FPluginState`; `session.get_sessions_info` reads them back under `currentSessionName`/`maxPlayers`/`isLANMatch` (emitting the honest "None"/0/false defaults only while `bConfigured` is still false, so it never serves a stale lie). `voiceChatEnabled` is now derived live from the `IVoiceChat` connection state (`IsInitialized() || IsConnected()`) instead of the hardcoded `false`, matching how `session.enable_voice_chat` operates `IVoiceChat`. Files: `Source/PinWright/Private/State/PluginState.h` (new `FConfiguredSessionSettings` struct + `ConfiguredSession()` accessor + field), `Source/PinWright/Private/State/PluginState.cpp` (reset in `ResetForTesting`), `Source/PinWright/Private/Handlers/System/SessionsHandler.cpp` (setter persists, getter reads back + live voice state, added `State/PluginState.h` include). Regression test `PinWright.session.get_sessions_info.ReadsBackConfiguredSettings` in `Source/PinWright/Private/Tests/EditorOps/TestSystemHandlers.cpp` configures `{sessionName:"CouchCoop", maxPlayers:4, bIsLANMatch:true}` then asserts `get_sessions_info` reads them back verbatim (and that an unconfigured getter still emits None/0); it fails on pre-fix code where the getter hardcoded the literals. `splitScreenType` / split-screen layout remain with sibling `E-session-info-split-screen-type-not-layout`; the echo-only voice setters (`set_voice_attenuation`/`configure_push_to_talk`) are out of scope here (their applied values are already confirmable via their own setter echoes).
- `#5-additional-voice-channel-readback` `IN-REVIEW` reporter — Additional evidence
  (new angle: the `activeVoiceChannels` field + `set_voice_channel` setter, which the
  `#4-fix` explicitly scoped OUT and prior encounters only listed but never demonstrated
  as a round-trip). A couch co-op proximity-voice task (REALISM mode, 2 local players,
  PIE running) set a named voice channel and then read back `get_sessions_info` to
  confirm it. Replay-confirmed live at HEAD: `session.set_voice_channel
  {channelName:"CoopPair", channelType:"Proximity"}` → `{"channelName":"CoopPair",
  "channelType":"Proximity"}` (pure echo, success); immediately after,
  `session.get_sessions_info {}` → `{"sessionsInfo":{...,"splitScreenType":
  "TwoPlayer_Horizontal","voiceChatEnabled":false,...,"activeVoiceChannels":[]}}` —
  `activeVoiceChannels` is `[]` despite the channel just being set, so the configured
  voice channel can never be read back. Source ground truth: `get_sessions_info`
  hardcodes an empty channel array (`SessionsHandler.cpp:821`-`822`: local
  `TArray<TSharedPtr<FJsonValue>> VoiceChannels;` set straight into
  `activeVoiceChannels` with nothing pushed), and `session.set_voice_channel`
  (`:601`-`624`) validates the type then echoes `channelName`/`channelType`, storing
  nothing. Note the `#4-fix` DID land for the sibling fields (this readback now shows
  `splitScreenType:"TwoPlayer_Horizontal"`, not `"Active"`, and `voiceChatEnabled` is
  live-derived — correctly `false` here since no voice provider is loaded, not a lie),
  but the `set_voice_channel`→`activeVoiceChannels` round-trip remains open. Same
  handler / same "setter echoes, getter hardcodes, round-trip can't close" failure mode
  this ticket owns; grouped here rather than filing a near-duplicate. The
  missing-voice-provider discoverability + false-"wired up" impression from the echo
  voice sub-setters is the process half, owned by `E-session-wiki-pie-prerequisite-undocumented`
  (`#2`/`#4`); not re-filed.
- `#6-additional-interface-type-no-field` `IN-REVIEW` reporter — Additional evidence
  (NEW ANGLE: the `interfaceType` session-identity field + `session.configure_session_interface`
  setter, which the two sibling `#5` history entries on `E-session-info-split-screen-type-not-layout`
  and this ticket both explicitly DELEGATED to this ticket as "the session-identity readback gap
  owned by `E-session-info-readback-hardcoded`" but which neither ticket body actually named until
  now). Surfaced by a LAN couch-coop setup task (focus `session.configure_session_interface`,
  outcome ergo). The success check explicitly wanted to read back the LAN interface being active
  before inviting anyone. Replay-confirmed live at HEAD (editor, PIE NOT running):
  `session.configure_session_interface {interfaceType:"LAN"}` → `{"interfaceType":"LAN","status":"configured"}`
  (echoes the enum), then `session.get_sessions_info {}` →
  `{"sessionsInfo":{"localPlayerCount":0,"inPlaySession":false,"currentSessionName":"CoopLAN","isLANMatch":true,"maxPlayers":4,"currentPlayers":0,"splitScreenEnabled":false,"splitScreenType":"TwoPlayer_Horizontal","voiceChatEnabled":false,"isHosting":false,"connectedServerAddress":"","activeVoiceChannels":[]}}`
  — the readback carries **no `interfaceType` field at all**, so the focus method's own effect is
  unobservable via the intended round-trip. LAN state is only inferable from `isLANMatch`, which is
  set by a DIFFERENT method (`configure_local_session_settings.bIsLANMatch`, now round-tripping post
  #4-fix — confirmed `isLANMatch:true` here), NOT by `configure_session_interface`. So a caller who
  only switches the interface to LAN without also setting `bIsLANMatch` gets zero readback
  confirmation. Source ground truth: `configure_session_interface` reads `interfaceType`
  (`SessionsHandler.cpp:143`), validates against Default/LAN/Null, echoes it back (`:154`), and
  stores it NOWHERE — a pure-echo setter exactly like the ones this ticket already owns; and
  `get_sessions_info` (`:778`-`819`) emits `currentSessionName`/`isLANMatch`/`maxPlayers`/
  `splitScreenEnabled`/`splitScreenType`/`voiceChatEnabled`/`isHosting`/`connectedServerAddress`/
  `activeVoiceChannels` but never an `interfaceType` field. Same handler / same "setter echoes,
  getter can't confirm" failure mode this ticket owns, one more session-identity field — grouped
  here (honoring the sibling tickets' explicit delegation) rather than a near-duplicate file. Fix
  extends the #4-fix pattern: persist the configured `interfaceType` into `FConfiguredSessionSettings`
  and add an `interfaceType` field to `get_sessions_info` (emitting the honest "Default" only while
  unconfigured), so the interface write→read round-trip closes. The task's other friction — the
  `configure_split_screen` `enabled:true` bit not round-tripping to `splitScreenEnabled` — is owned by
  `E-session-info-split-screen-type-not-layout` (`#5-additional-enabled-boolean-no-roundtrip`, this
  exact task's evidence); the `add_local_player [NO_GAME_INSTANCE]` on this PIE-stopped host is the
  expected/task-excluded prerequisite owned by `E-session-wiki-pie-prerequisite-undocumented`.
  Neither re-filed here.
