---
id: E-add-sync-marker-frame-vs-seconds
title: "animation.authoring.add_sync_marker writes by integer `frame` while list_sync_markers/describe_sequence read markers back in seconds — the read/write unit pairing is undocumented, forcing a manual seconds->frame conversion"
status: OPEN
severity: Low
category: ergonomic
tags: [write-read-unit-asymmetry, units, animation, add_sync_marker, docs, discoverability]
encounters: 1
lastSeen: 2026-07-10T23:32:49.0598197+03:00
---

# add_sync_marker takes a frame integer, its readers report seconds, and the docs never pair the two

`animation.authoring.add_sync_marker` places a sync marker by an integer `frame`
(wiki: `frame (number, optional): Frame number (default 0)`). But every reader of
the same markers reports position in **seconds**: `animation.authoring.list_sync_markers`
returns entries like `{"name":"LeftFootDown","time":0.4333333}` and
`animation.describe_sequence`'s `syncMarkers[]` are likewise `time` floats.

The two units are each documented in isolation, but nothing pairs them: a caller
who reads an existing marker's time in seconds (or, as here, reads a content
foot-contact time in seconds) and wants to add a phase-aligned marker at that same
instant has to work out the `frame = round(seconds * frameRate)` conversion
themselves. This works — it is not a defect (the `frame` param is documented and
add_sync_marker succeeded first try) — but the seconds->frame step is non-obvious
and unsupported by the docs, and it quantizes to the frame grid (no way to express
a sub-frame time the reader just handed back).

This is the write-side member of the same clip's sync-marker read/write pair:
`list_sync_markers` (the reader, `F-rpc-animation-list-sync-markers` DONE) and
`describe_sequence` both speak seconds; only the writer speaks frames.

## What it should do

Document the pairing on the `add_sync_marker` overlay in
`docs/wiki-src/animation.authoring.md`: state that markers are written by integer
`frame` while `list_sync_markers` / `describe_sequence` report each marker's `time`
in **seconds**, and give the explicit `frame = round(time * frameRate)` conversion
(with a note that placement snaps to the display-frame grid). Optionally cross-link
the reader H3s. (A stronger, out-of-scope-for-docs option the CallAnalyzer floated
is to also accept a `time`/`timeSeconds` param on add_sync_marker so read and write
speak the same unit — noted here for the fix owner, not required by this ticket.)

## Evidence

Struggle audit of a clean/done locomotion sync-group task (focus
`animation.authoring.list_sync_markers`; place LeftFootDown/RightFootDown
foot-plant markers on `MM_Walk_Fwd_RL_Off` and `MM_Run_Fwd_RL_Off` under
`/Game/ExampleContent/Sequencer/Animations`, then read them back). 18 MCP calls,
zero retries, judge filed nothing on the outcome for this method (it filed the
neighbor `E-describe-sequence-no-compact-mode` for the describe_sequence overflow,
and dismissed the frame-vs-seconds asymmetry as documented convention).

Both clips are 30fps, so the write frames landed on read-back seconds exactly:
Walk `LeftFootDown frame=13` read back at `0.433s`, `RightFootDown frame=27` at
`0.900s`; Run `LeftFootDown frame=7` at `0.233s`, `RightFootDown frame=16` at
`0.533s`. To match a content foot-contact the agent read in seconds
(`describe_sequence` marker `L` at `0.4168641s`) it converted to `frame 13`
(= `0.4333333s`), a ~17ms snap to the 30fps grid.

Agent friction line, verbatim: "add_sync_marker takes an integer frame (not
seconds), so I converted the content's true foot-contact times to nearest frames."
CallAnalyzer (ground-truth trace): "Read-side reports marker positions in SECONDS
... But the write-side add_sync_marker takes only an integer `frame` ... the agent
had to convert seconds to the nearest frame ... quantizing to the 30fps grid."

Same board family as the write-side unit/shape discoverability tickets
`E-volume-set-extent-units-class-dependent-docs`,
`E-volume-set-bounds-flat-array-vs-minmax-object`, and
`E-material-connect-nodes-target-input-arg-asymmetry` — a write verb whose
unit/shape differs from its paired reader and needs a docs pairing note. Distinct
from `E-sequencer-property-unit-drift` (that is a sequencer frames-vs-ticks WRITE
bug — a silent ~1000x wrong range; here nothing is wrong, the units are just
un-paired in the docs).

severity rationale: impact=docs/discoverability (Low) × reach=anim sync-marker
authoring is a specific path, not every-session -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a clean/done locomotion sync-group task (focus `animation.authoring.list_sync_markers`; 18 MCP calls, zero retries, outcome ergo). add_sync_marker writes by integer `frame` while its paired readers `list_sync_markers` and `describe_sequence` report each marker's `time` in seconds; the two units are documented separately but never paired, so placing a marker at a time you read in seconds forces a manual `frame = round(time*frameRate)` conversion that snaps to the frame grid. Not a defect (the `frame` param is documented and the call worked first try; judge dismissed the ergonomic-defect framing as documented convention) — filed as the discoverability residual: a docs pairing note on `docs/wiki-src/animation.authoring.md`'s add_sync_marker H3 giving the seconds<->frame relationship. Evidence: agent converted content marker `L@0.4168641s` to `frame 13` (`0.4333s`, ~17ms grid snap); friction line "add_sync_marker takes an integer frame (not seconds), so I converted the content's true foot-contact times to nearest frames." Same write-side-unit/shape-docs family as `E-volume-set-extent-units-class-dependent-docs` / `E-volume-set-bounds-flat-array-vs-minmax-object` / `E-material-connect-nodes-target-input-arg-asymmetry`.
