---
id: E-texture-streaming-priority-boolean-only
title: "texture.set_streaming_priority name promises graduated priority but only toggles boolean neverStream — 'nudge priority down' has no faithful mapping"
status: OPEN
severity: Low
category: ergonomic
tags: [texture, streaming, never-stream, misleading-name, neverStream, docs]
encounters: 2
costly: 2
lastSeen: 2026-06-23T10:08:23Z
---

# `texture.set_streaming_priority` is named for graduated priority but its only knob is a boolean `neverStream`

`texture.set_streaming_priority` (registered "Set streaming priority",
`TextureHandler.cpp:2833`) accepts exactly one functional parameter:
`neverStream: boolean` (`TextureHandler.cpp:1195-1245`,
`ValidParams = {subAction, assetPath, neverStream, save}`). It just assigns
`Texture->NeverStream = bNeverStream` and reports success message
`"Streaming priority configured"`.

The **method name and summary promise a graduated streaming priority**, but the
only available action is a binary all-or-nothing switch that *disables mip
streaming entirely* for the texture. There is no parameter for a relative
priority / streaming-distance multiplier / a "lower this texture's streaming
importance" nudge. So when a caller's intent is the natural graduated one —
here: *"the detail noise is low-priority for streaming, so nudge its streaming
priority down"* — there is **no faithful mapping**. The attempt was forced to
use `neverStream=true` as a proxy for "de-prioritize," which is semantically
much stronger and arguably wrong: `neverStream=true` doesn't *lower* streaming
priority, it *removes the texture from the streaming system* (forces it fully
resident / non-streamed), the opposite of "drops mips sooner on lower settings."

Two coupled problems:
1. **Misleading name** — `set_streaming_priority` + "Set streaming priority"
   implies a graduated scalar; the surface only has a boolean kill switch. A
   caller reasonably reaches for it expecting to nudge priority and instead
   silently makes a much larger, near-opposite change.
2. **Success message overstates the action** — `"Streaming priority configured"`
   reads as if a priority value was set, masking that all that happened was a
   `NeverStream` flip.

## What it should do

Either (a) **rename/re-scope** to reflect reality (e.g. summary "Toggle mip
streaming (neverStream)" and message `"neverStream set to <v>"`), **or**
(b, preferred for the stated intent) **add a graduated knob** — e.g. a
`priorityBias` / `streamingPriority` scalar or a documented mapping onto the
engine's actual streaming-priority levers (`LODBias` toward earlier mip drop,
`GlobalForceMipLevelsToBeResident`, or a streaming-distance multiplier) so that
"de-prioritize streaming" has a non-destructive path that doesn't require the
caller to abuse the all-resident `neverStream` flag.

Docs follow-up (downstream wiki process): `docs/wiki-src/texture.md` documents
none of the setters — it should state plainly that `set_streaming_priority`
today only toggles `neverStream` (no graduated priority) and that `neverStream`
forces the texture *resident*, not lower-priority, so callers don't misuse it for
"drop mips sooner."

**Workaround:** to make a texture drop mips sooner on lower settings, use
`texture.set_lod_bias` (positive bias) rather than `set_streaming_priority`;
reserve `neverStream=true` for "must stay fully resident," which is the opposite
of de-prioritizing.

## Evidence (from the task friction note)

> "the streaming-priority helper only toggles a boolean `neverStream` (no
> graduated priority)"

Self-report shows the proxy misuse: the run set
`set_streaming_priority neverStream=true` and described it as
"`neverStream=true` to de-prioritize streaming" — but `neverStream` forces
residency, so the chosen proxy is semantically backwards relative to the user's
"nudge priority down / drops mips sooner" intent. The genuinely
intent-matching action in the same run was the separate `set_lod_bias lodBias=1`.
Source: `TextureHandler.cpp:1195-1245` (only `neverStream`), `:2833-2836`
(name/summary/param doc).

## History
- `#1-initial-audit` `OPEN` reporter — Configuring a low-streaming-priority
  detail-noise texture, the attempt found `set_streaming_priority` exposes only a
  boolean `neverStream` and used `neverStream=true` as a stand-in for "lower
  priority" — a near-opposite action, since `neverStream` forces residency rather
  than dropping mips. PROCESS friction: misleading method name/summary/success-
  message ("Set streaming priority" / "Streaming priority configured") for what
  is a binary mip-streaming kill switch, with no graduated priority knob for the
  common "de-prioritize streaming" intent. Source-confirmed
  `TextureHandler.cpp:1195-1245` / `:2833`. Fix: rename to reflect the boolean,
  or add a graduated knob; tag `docs` to clarify on `docs/wiki-src/texture.md`
  that `neverStream` forces residency (not lower priority) and that mip-drop is
  `set_lod_bias`.
- `#2-evidence-raise-priority-intent` `OPEN` reporter — Second independent task
  hits the same wall from the *opposite* intent direction (raise, not lower):
  optimizing `T_GrungySurface_slnneipc_4K_MR` (4K mask map), the user asked to
  "set its streaming priority to a higher value via texture.set_streaming_priority."
  The run found the method "exposes no numeric priority, only a neverStream boolean,
  so 'set streaming priority to a higher value' is not expressible" — and called it
  with `neverStream:false` (a no-op-ish toggle), then could only report success on
  the proxy, not the requested graduated value. Confirms the misnamed-method
  friction is bidirectional: neither "nudge down" nor "raise to a higher value" has
  a faithful mapping onto the lone boolean. 10-call task (incl. baseline + post + 2
  final describes); same source surface `TextureHandler.cpp:1195-1245`/`:2833`.
