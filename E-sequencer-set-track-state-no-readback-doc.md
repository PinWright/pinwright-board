---
id: E-sequencer-set-track-state-no-readback-doc
title: "sequencer.set_track_* wiki gives no signal the write is unverifiable — callers waste fallback read probes"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, sequencer, readback, round-trip]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `sequencer.set_track_*` wiki gives no signal the write is unverifiable — callers waste fallback read probes

This is the **process/docs** companion to `F-sequencer-track-state-readback` (the
feature that would add the missing read surface). That ticket fixes the
capability; this one is about the discovery cost a caller pays *today*, before
any fix lands.

`docs/wiki-src/sequencer.md` documents none of `set_track_solo`,
`set_track_muted`, `set_track_locked`, or `list_tracks`, and gives no warning
that the per-track mute/solo/lock flags these setters write are **not surfaced by
any sequencer read RPC**. A caller following the natural "write then read back to
verify" loop has no way to know, from the docs, that the verification step is
impossible — so they discover it the expensive way: by probing one read surface
after another until they give up.

The wiki note is a cheap, durable mitigation independent of whether/when the
`F-` feature lands: it sets the expectation up front ("these writes succeed but
have no MCP read-back; do not budget a verification round-trip") and names the
`python.execute` / `property.get` fallback so a caller that *needs* to verify
goes straight to it instead of cycling through `list_tracks` → `list_sections` →
`get_metadata`.

**What to add to the overlay** (`docs/wiki-src/sequencer.md`): a short
note on the `set_track_solo` / `set_track_muted` / `set_track_locked` entries
stating that the resulting state is not readable through any `sequencer.*` read
RPC (`list_tracks` / `list_sections` / `get_metadata` all omit it), that solo is
*simulated* by eval-disabling every other track (so there is no distinct "solo"
flag), and that the only read-back today is `python.execute`
(`Track.is_eval_disabled()` / `Section.is_locked()`) or `property.get` on
`bIsEvalDisabled` / `bIsLocked`. Cross-link `F-sequencer-track-state-readback`
so the note can be tightened/removed once the read surface exists.

**Evidence (this task):** task focus `sequencer.set_track_solo`, outcome `gap`.
Friction note verbatim: "the struggle was verification — the designated readback
surface (sequencer.list_tracks) returns no mute/lock/solo fields, forcing me to
probe list_sections (378KB, dumped to disk) and get_metadata as fallbacks, none
of which expose the state." Call-log: all three `set_track_*` writes plus the
two reads (`get_properties`, `list_tracks`) succeeded first try with no retries —
**method discovery was smooth; the wasted calls were all verification probes**
(`list_tracks` readback, then `list_sections` and `get_metadata` as fallbacks =
3 reads spent confirming a round-trip that the docs could have flagged as
unavailable in one line).

## History
- `#1-initial-audit` `OPEN` reporter — Process/docs companion to `F-sequencer-track-state-readback`. `docs/wiki-src/sequencer.md` documents neither the `set_track_*` writers nor that their state is unreadable via any `sequencer.*` read RPC, so callers following write-then-verify discover the gap by probing `list_tracks` → `list_sections` (378KB) → `get_metadata`. Evidence: this task's writes succeeded first-try (no discovery friction); the 3 wasted calls were all verification probes. Proposed a one-line wiki note on the setter entries (no readback exists; solo is simulated; fallback is `python.execute`/`property.get`) cross-linking the `F-` feature.
