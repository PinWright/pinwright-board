---
id: E-sequencer-get-metadata-identity-only
title: "sequencer.get_metadata returns only asset identity {path,name,class} — the frame rate / playback range / binding counts its name implies live in sequencer.get_properties, so a caller wanting sequence 'metadata' pays an extra get_properties round-trip; the wiki page documents no return fields"
status: OPEN
severity: Low
category: ergonomic
tags: [sequencer, get_metadata, get_properties, readback, round-trip, naming, docs]
encounters: 1
lastSeen: 2026-07-02T09:55:31.2017475+03:00
---

# `sequencer.get_metadata` returns only identity — authored rate/range/bindings live in `get_properties`

`sequencer.get_metadata` is the natural "tell me about this sequence" call, and
its name implies rich sequence metadata. In practice it returns only **asset
identity** — `{path, name, class}` (e.g. `{"path":".../IntroMaster",
"name":"IntroMaster","class":"LevelSequence"}`) — and carries **none** of the
authored MovieScene state a caller usually means by "metadata": no frame rate, no
tick resolution, no playback range, no binding counts. That data is only returned
by the separate `sequencer.get_properties` (`{frameRate, playbackStart,
playbackEnd, duration, bindingCount, …}`).

So a caller that wants to confirm a sequence's rate/range calls `get_metadata`
first (by name), gets back only identity, and must issue a second
`get_properties` call to get the numbers — an avoidable back-to-back round-trip.
Compounding it, the **`sequencer.get_metadata` wiki page documents no return
fields at all**, so there is nothing that steers the caller to `get_properties`
in the first place; the sparse-vs-rich split is only learned by calling both.

This is the same naming/readback family as `E-get-blackboard-value-omits-value`
("the verb's name promises a value its readback never carries; document the other
route") — here the verb's name ("metadata") promises rich sequence data its
readback never carries, and the fix is to point callers at `get_properties`.

`get_metadata` is genuinely useful as a **lightweight existence/identity probe**
(the audited task also used it that way to verify two deleted drafts returned
`[INVALID_SEQUENCE] Sequence not found`), so the point is not that it is broken —
it's that the name over-promises and the wiki documents nothing, forcing the
extra call.

## What to do

- **Docs (primary, cheap — downstream wiki process):** on the
  `sequencer.get_metadata` page in `docs/wiki-src/sequencer.md`, document exactly
  which fields it returns (`path`, `name`, `class`) and state that authored scene
  metadata — frame rate, tick resolution, playback range, binding/spawnable/
  possessable counts — comes from `sequencer.get_properties`, so a caller wanting
  the numbers goes straight there and skips the wasted first call. (`get_properties`
  itself was recently extended with tickResolution + counts per DONE
  `E-rpc-sequencer-extend-get-properties`, so it is the correct one-stop richer
  reader to name.)
- **Optional ergonomic:** have `get_metadata` also echo `frameRate` +
  `playbackRange` (a couple of cheap fields already on the MovieScene) so the
  common "what's this sequence's rate/range?" check is satisfied in one call — or
  explicitly keep it identity-only and let the docs do the routing.

**Workaround (today):** for anything beyond path/name/class, skip `get_metadata`
and call `sequencer.get_properties` directly (rate, range, duration, binding
counts); use `get_metadata` only as a fast existence/identity probe.

## Distinct from

- `E-rpc-sequencer-extend-get-properties` (DONE) — that *extended `get_properties`*
  with tickResolution/spawnable/possessable counts (dump-vs-live parity). This
  ticket is about the *other* reader, `get_metadata`, whose name implies that
  richer data but returns only identity, and about the missing wiki routing
  between the two.
- `E-get-blackboard-value-omits-value` (IN-REVIEW) — the same naming/readback
  family on a different namespace (a verb whose name promises a value its readback
  omits; document the alternate route). Cited as the sibling pattern.
- `E-sequence-add-keyframe-bare-empty-no-success` /
  `E-sequence-add-keyframe-per-axis-value-shape-undocumented` (OPEN) — other
  sequencer readback/response-shape gaps on `add_keyframe`, unrelated method.

## Evidence

Struggle-audit (PROCESS) of the SEED-mode `sequencer.delete` cinematic task
(focus `sequencer.delete`, namespace `sequencer`, `/Game/Cinematics/IntroMaster`,
23 calls, outcome done; judge filed the add_keyframe bare-`{}` gap). The
CallAnalyzer flagged this as a `surprising` inefficiency: `get_metadata
{path:/Game/Cinematics/IntroMaster}` returned only
`{"path":"…IntroMaster","name":"IntroMaster","class":"LevelSequence"}` — no frame
rate, no playback range — so the Attempt immediately issued a second
`get_properties` call to obtain `{frameRate:30/1, playbackStart:0,
playbackEnd:90, duration:90, bindingCount:1}`. The task's success check itself
assumed `get_metadata` would reflect the authored 30fps + range (per
`plan_divergence`: "Prep assumed get_metadata … would be the metadata proof
reflecting 30fps+range, but get_metadata returned only {path,name,class}, so the
Attempt needed an extra get_properties call"). The `sequencer.get_metadata` wiki
page currently documents no return fields, so nothing warned that the rate/range
live in `get_properties`.

severity rationale: impact=discoverability/naming + a readback that omits the
fields the caller expected (one extra `get_properties` call, no wrong/lie data)
× reach=rare (occasional sequencer inspection, not an every-session method) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle-audit of the
  SEED-mode `sequencer.delete` cinematic task (`/Game/Cinematics/IntroMaster`, 23
  calls; judge filed `E-sequence-add-keyframe-bare-empty-no-success`). Distinct
  PROCESS angle (CallAnalyzer `surprising`): `sequencer.get_metadata` returned
  only `{path,name,class}` while the authored frame rate / playback range / binding
  count live in `sequencer.get_properties`, forcing a back-to-back `get_properties`
  call; the plan itself assumed `get_metadata` would carry the 30fps+range proof.
  The method's name over-promises and the `sequencer.get_metadata` wiki page
  documents no return fields, so nothing routes the caller to `get_properties`.
  Proposed: document the sparse identity return + point to `get_properties` on the
  `sequencer.get_metadata` page in `docs/wiki-src/sequencer.md` (optionally echo
  `frameRate`+`playbackRange` from `get_metadata`). Dedup: ripgrep across
  OPEN/IN-REVIEW/DONE/WONTFIX — no ticket names `get_metadata`; distinct from
  `E-rpc-sequencer-extend-get-properties` (DONE; that extended the *other* reader),
  same naming/readback family as `E-get-blackboard-value-omits-value` (IN-REVIEW).
