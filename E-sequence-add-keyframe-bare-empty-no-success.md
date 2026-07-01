---
id: E-sequence-add-keyframe-bare-empty-no-success
title: "sequence.add_keyframe (frame form) returns a bare {} with no success field, forcing a list_sections readback to confirm the key landed"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, sequencer, add_keyframe, success-field, readback, round-trip]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `sequence.add_keyframe` (frame-numbered form) returns a bare `{}` — no `success` field to confirm the write

The frame-numbered `sequence.add_keyframe` writer (the legacy `SequenceHandler`
form used for transform/vector/object-typed tracks) returns an **empty JSON
object `{}`** on success — no `success: true`, no echo of the frame/property/
channel(s) written, no key count. Every other authoring verb on the board's
convention returns at least a `success` flag and usually an echo of what changed,
so a caller cannot tell from the `add_keyframe` response whether the key actually
landed. The natural reaction — and the one the audited task took — is to spend an
extra `sequencer.list_sections` readback purely to confirm the keyframe exists.

This is the response-shape ergonomic gap, distinct from two neighbor tickets on
the same method:

- `B-sequence-add-keyframe-location-property-rejected` (IN-REVIEW) is about
  `property="Location"` being *rejected*; this ticket is about the *successful*
  `property="Transform"` path returning a bare `{}`.
- `F-sequencer-transform-section-range-not-expanded-by-keyframe` (OPEN) is about
  the section range staying collapsed; this ticket is about the missing success
  confirmation that *forces the readback in the first place*.

The bare-`{}` return also defeats normal success detection: a generic caller that
checks for a truthy `success` field sees none and can't distinguish success from
a silent no-op without a separate read.

## Evidence (this task)

SEED-mode `IntroFlythrough` cinematic task (`sequencer.create` focus, 17 calls).
Two `sequence.add_keyframe` calls (Transform property, frames 0 and 120) each
`ok=true` but each "returned bare `{}` - no success field" (per-call error_text),
which the run noted forced a verification readback. Verbatim friction note: "it
returned a bare {} with no success confirmation, forcing a list_sections readback
to verify keys landed." All calls succeeded; the friction is the missing
confirmation field, not a failure.

## What to do

1. **Emit a success/echo payload (primary).** Have the frame-numbered
   `sequence.add_keyframe` return at minimum `{ success: true }`, ideally with an
   echo of what was written (e.g. `property`, `frame`/`tickFrame`, the channel
   indices touched, and/or a per-channel `keyCount`), matching the board's
   authoring-response convention so the caller doesn't need a `list_sections`
   round-trip to confirm. The modern seconds-based `SequencerHandler` form's
   response shape is the reference to align to.
2. **Wiki note (downstream wiki process, cheap interim).** On the
   `sequencer.add_keyframe` H3 in `docs/wiki-src/sequencer.md`, state that the
   frame-numbered form currently returns a bare `{}` on success (no `success`
   field) and that confirmation today requires a `sequencer.list_sections`
   readback — so a caller budgets it instead of treating the empty response as a
   failure. Tighten/remove once (1) lands.

**Workaround (today):** treat a bare `{}` from the frame-numbered
`sequence.add_keyframe` as success, and if you must verify, read back with
`sequencer.list_sections` (check the target channel's `keyCount`).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of the SEED-mode
  `IntroFlythrough` cinematic task (`sequencer.create` focus, 17 calls; judge
  filed the section-range gap). This ticket covers a distinct PROCESS surface
  from the friction note: the frame-numbered `sequence.add_keyframe`
  (`property="Transform"`) returned a bare `{}` with no `success` field on both
  keyframe calls (frames 0 and 120), forcing a `sequencer.list_sections` readback
  to confirm the keys landed. Dedup: ripgrep across OPEN/DONE/WONTFIX — distinct
  from `B-sequence-add-keyframe-location-property-rejected` (IN-REVIEW; that's the
  `property="Location"` *rejection*, this is the *successful* path's empty
  response) and from `F-sequencer-transform-section-range-not-expanded-by-keyframe`
  (OPEN; that's the section range, this is the missing success confirmation that
  triggers the readback). The wiki `add_keyframe` H3 documents the dual call
  shape but never the bare-`{}` return. Sibling docs/process angle to
  `E-sequencer-set-track-state-no-readback-doc` (wasted verification probes from
  an unsignalled read-gap). Primary ask: emit `success`/echo from the
  frame-numbered writer; interim: a wiki note on the `add_keyframe` H3.
