---
id: E-describe-sequence-no-compact-mode
title: "animation.describe_sequence has no compact/field-select mode — the boneTracks[] array (added for bone-key readback) overflows the 10000-char display limit on every real skeletal clip and spills to disk"
status: OPEN
severity: Low
category: ergonomic
tags: [animation, describe_sequence, get-animation-info, response-size, oversized-readback, compact, response-spill, docs]
encounters: 3
lastSeen: 2026-07-09T11:53:22.8851331+03:00
---

# describe_sequence dumps the full per-bone boneTracks array by default, so every read of a real clip spills to disk

`animation.describe_sequence` (top-level, shipped by `E-dump-rpc-parity` `#3`/`#4`
as the dump-parity live read for `UAnimSequence`) is named and documented as a
**compact metadata reader**. Its verified original shape (`E-dump-rpc-parity` `#4`)
was small and inline: `{assetKind, path, lengthSeconds, frameRate, numFrames,
notifies, curves, syncMarkers}`. The wiki advertises it the same way — "length,
frame rate, additive type, skeleton, notifies, curves, sync markers".

Since then `E-rpc-animation-bone-track-readback` (IN-REVIEW) wired a full
per-bone **`boneTracks[]`** array (one `{boneName, keyCount}` entry per raw bone
track) into `BuildAnimSequenceJson`, so `describe_sequence` now carries it too.
That addition is correct and wanted — it lets `set_bone_key` / `add_bone_track`
writes be verified live — but on **any real skeletal-mesh clip** the boneTracks
array is one entry per skeleton bone, and that alone pushes the response past the
**10000-char MCP display limit**, so every `describe_sequence` call on a real clip
is written to a `Saved/.../HttpResponses/<uuid>.json` file that the agent must
then `Read` out-of-band to extract a value as small as `lengthSeconds` or the
`notifies` array.

There is no `compact`, `headerOnly`, `includeBoneTracks`, or field-select
parameter — it is all-or-nothing, so the common "how long is this clip / does it
have my notify?" read pays a disk round-trip it never needed.

## Evidence

Struggle audit of a clean/done locomotion task (build `ABP_Mannequin_Locomotion`
+ `BS_Mannequin_Movement` on `SK_Mannequin_Skeleton`, add a `Footstep` notify to
`Walk_Fwd`; every call first-try, zero retries, judge filed nothing on the
outcome). Every `describe_sequence` against a real 68-bone Mannequin clip
overflowed the display threshold and spilled to disk, forcing an extra Read —
**3 overflow+Read round-trips in one task**:

- `Walk_Fwd` pre-check (length/skeleton) — **11214 chars**
- `Jog_Fwd` pre-check (length/skeleton) — **12432 chars**
- `Walk_Fwd` re-read to verify the Footstep notify landed — **11464 chars**

CallAnalyzer note, verbatim: the payload "is dominated by a full per-bone
boneTracks array (68 entries, each with keyCount) plus rawTrackCount that the task
never used; the length/skeleton/notifies fields the agent actually wanted are
tiny." The task only ever wanted lengths (to size the blend space) and the notify
list — every field it read was a few bytes; the boneTracks dump it never used is
what tipped the response over.

## Also affects `animation.authoring.get_animation_info` (same `boneTracks[]` source)

The identical overflow reproduces on the sibling introspection reader
`animation.authoring.get_animation_info`. `E-rpc-animation-bone-track-readback`
(IN-REVIEW) wired the same per-bone `boneTracks[]` array (from the shared
`BuildBoneTracksArrayJson` helper) into the `get_animation_info` `UAnimSequence`
branch too — so a plain "duration + skeleton" lookup on a normally-keyed clip
overflows the exact same 10000-char display limit and spills to a
`HttpResponses/<uuid>.json` sidecar. The fix therefore has to guard the
`boneTracks[]` inlining in **both** readers, not just `describe_sequence`.

Struggle audit of a clean/done composite build-out (focus
`animation.authoring.add_composite_segment`; the seed verb worked and the judge
filed the composite-thinness neighbor `E-get-animation-info-thin-on-anim-composite`;
13 MCP calls, outcome ergo). To compute the stitched composite's total length the
task needed only each source clip's `duration`. `get_animation_info` on `Dino_Idle`
fit inline (`rawTrackCount 0`, `boneTracks []`), but the same call on `Dino_Walk`
returned `outputTooLong` at **11204 chars** (threshold 10000) and spilled to disk,
forcing a `Grep` + `Read offset 1 limit 25` just to recover the single `duration`
float. Two clips of identical length (both 30f@30fps) behaved inconsistently
purely because `Dino_Walk` carries populated raw bone tracks and `Dino_Idle` does
not — the `boneTracks[]` array is the sole overflow source, exactly as on
`describe_sequence`.

Whichever `compact`/`includeBoneTracks`/`fields` knob lands must also guard the
`boneTracks[]` inlining in the `get_animation_info` `UAnimSequence` branch
(`AnimationAuthoringHandler_Sequence.cpp`), and `docs/wiki-src/animation.authoring.md`
(the `get_animation_info` overlay) needs the same boneTracks note as
`docs/wiki-src/animation.md`.

## This is the `oversized-readback` family — not "remove boneTracks"

This is the same shape the board has already fixed for the other structural-dump
readers, and `describe_sequence` is the anim-sequence member of that family with
no size knob:

- [`E-describe-metasound-no-compact-mode`](E-describe-metasound-no-compact-mode.md)
  (IN-REVIEW) — added `compact` / `nodeIds` so `describe_metasound` no longer
  spills on every graph readback. The exact analog: a describe RPC whose default
  full payload overflows the 10000-char limit even on trivial assets.
- [`E-widget-export-xml-token-limit`](E-widget-export-xml-token-limit.md) (DONE)
  — `compact` / `omit_slot_chain` on `widget.export_xml`.
- [`E-graph-connections-pagination`](E-graph-connections-pagination.md) (DONE)
  — `nodeIds` / `maxEdges` on `get_graph_connections`.

**Important — the fix is NOT to drop boneTracks by default.** A CallAnalyzer
hypothesis for this task suggested omitting `boneTracks`/`rawTrackCount` from the
default response; doing so would regress the just-shipped
`E-rpc-animation-bone-track-readback` (IN-REVIEW), which added `boneTracks[]`
specifically so hand-keyed bone writes are live-verifiable (no other reader
surfaces them). The family-consistent fix keeps the full snapshot as the default
and adds an opt-in size knob for the metadata-only read, e.g. one or more of:

- `compact: true` (default `false`, existing callers unchanged) — drop the
  `boneTracks[]` array (and per-track detail) and keep the tiny metadata:
  `lengthSeconds` / `frameRate` / `numFrames` / skeleton / `notifies` / `curves`
  summary + a `boneTrackCount` header. This alone keeps the routine
  length/notify read well under the display threshold.
- a `fields` / `include` projection selector so a caller can ask for exactly
  `lengthSeconds`+`notifies`.

Default behavior stays the full snapshot (parity with `asset.dump`'s
`anim_sequence.json` sidecar and the bone-track readback ticket); only the
inline-metadata read opts into the compact form.

`E-http-response-spill` (DONE) does NOT cover this: it scopes the server-side
file-reference fallback to direct-HTTP callers and explicitly bypasses
MCP-adapter calls (MCP clients own large-output behavior) — here the caller was
on the MCP path, so the spill came from the MCP client's own large-output
handling, exactly the case the per-method compact modes above exist to avoid.

## Docs angle

The `describe_sequence` overlay (`docs/wiki-src/animation.md`) still advertises
only "length, frame rate, additive type, skeleton, notifies, curves, sync
markers" — it never mentions the now-dominant per-bone `boneTracks[]` array, so a
caller has no warning that a "metadata" read overflows on every real clip.
Whichever way the size knob lands, `docs/wiki-src/animation.md` should document
the `boneTracks[]` field and, once a `compact`/projection param exists, recommend
it for metadata-only reads (mirroring the `describe_metasound` `#4` docs
recommendation).

## Distinct from the other describe_sequence tickets

- `E-rpc-animation-bone-track-readback` (IN-REVIEW) — *added* `boneTracks[]` for
  readback; this is the follow-on that the always-full array is what overflows the
  inline limit. A size knob on top of that feature, not a regression of it.
- `E-animation-sequence-info-reader-name-split` (IN-REVIEW) — method-*name*
  discoverability (`animation.get_animation_info` UNKNOWN_ACTIONs; the reader is
  `describe_sequence`). Orthogonal: naming, not payload size.
- `E-dump-rpc-parity` (DONE) — *added* the RPC and its baseline shape; this is the
  size follow-on.

**Workaround:** read the spilled `HttpResponses/<uuid>.json` off disk each call;
the length/notifies values are in there, just not inline.

severity rationale: impact=response-spill that only forces a `Read` (Low) ×
reach=common on anim-setup tasks but not every-session -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a clean/done locomotion task (build `ABP_Mannequin_Locomotion` + `BS_Mannequin_Movement` + Footstep notify on `SK_Mannequin_Skeleton`; 22 MCP calls, zero errors, zero retries, judge filed nothing on the outcome). Every `animation.describe_sequence` against a real 68-bone Mannequin clip overflowed the 10000-char display limit and spilled to disk, forcing an extra out-of-band `Read` — 3× in the one task (Walk_Fwd pre-check 11214 chars, Jog_Fwd pre-check 12432 chars, Walk_Fwd notify-verify re-read 11464 chars). Root cause: the per-bone `boneTracks[]` array (added by `E-rpc-animation-bone-track-readback` IN-REVIEW for bone-key readback) is one entry per skeleton bone and dominates the payload, while the fields the task actually used (lengths, notifies) are tiny; there is no `compact`/`headerOnly`/field-select knob. Same oversized-readback family as `E-describe-metasound-no-compact-mode` (IN-REVIEW), `E-widget-export-xml-token-limit` (DONE), `E-graph-connections-pagination` (DONE); NOT covered by `E-http-response-spill` (DONE, direct-HTTP-only). Proposed fix: opt-in `compact:true` (drop boneTracks[], keep length/frameRate/skeleton/notifies/curves + a boneTrackCount header) and/or a `fields` projection, default unchanged — explicitly NOT removing boneTracks by default (that would regress the bone-track readback ticket). Docs: `docs/wiki-src/animation.md` should document the boneTracks[] field and recommend the compact param for metadata-only reads.
- `#2-additional-get-animation-info` `OPEN` auditor — Same `boneTracks[]`-overflow root cause reproduced on the SIBLING reader `animation.authoring.get_animation_info` (not just top-level `describe_sequence`) — added it as a co-affected method rather than filing a near-duplicate. Struggle audit of a clean/done composite build-out (focus `animation.authoring.add_composite_segment`, 13 MCP calls, outcome ergo; judge filed the composite-thinness neighbor `E-get-animation-info-thin-on-anim-composite`): computing the stitched composite's length needed only each source clip's `duration`, but `get_animation_info` on `Dino_Walk` overflowed the 10000-char display limit at **11204 chars** and spilled to a HttpResponses sidecar, forcing a `Grep` + `Read offset 1 limit 25` to recover one float; the same call on `Dino_Idle` (`rawTrackCount 0`, empty `boneTracks[]`) fit inline — proving the per-bone `boneTracks[]` array (added to `get_animation_info`'s `UAnimSequence` branch by `E-rpc-animation-bone-track-readback` IN-REVIEW, same `BuildBoneTracksArrayJson` source as `describe_sequence`) is the sole overflow. Same fix: the `compact`/`includeBoneTracks` knob must guard `boneTracks[]` inlining in BOTH `describe_sequence` and `get_animation_info`; docs note also needed on `docs/wiki-src/animation.authoring.md`. encounters 1->2.
- `#3-liveness` `OPEN` auditor — Same `boneTracks[]`-overflow on `animation.authoring.get_animation_info` still observed; pure reconfirm of `#2` (same method, root cause, and fix). Clean/done footstep-notify task (focus `animation.authoring.add_notify`; judge filed the notify-state startFrame drop on the neighbor `B-add-montage-notify-time-dropped`, culprit `add_notify_state`). Both canonical UE Mannequin locomotion clips overflowed in the one task: `get_animation_info` on `MM_Run_Fwd` (58f, fully-keyed ~58-bone rig) returned `outputTooLong` at **15922 chars** (threshold 10000) and `MM_Walk_Fwd` (83f) likewise, each spilling to a HttpResponses sidecar the agent had to Read just to recover the scalar `duration`/`numFrames`/`frameRate` — 2 spill+Read round-trips for clip timing in a single task, on stock Mannequin content. No new angle. encounters 2->3.
