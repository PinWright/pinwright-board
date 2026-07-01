---
id: F-recorder-segment-windows
title: "Recorder: first-class segment windows (race round, lobby, sumo round, editor session)"
status: IN-REVIEW
severity: Medium
category: feature
tags: [recorder, telemetry, ergonomics]
---

# Recorder: first-class segment windows (race round, lobby, sumo round, editor session)

The `recorder.*` query surface has no notion of a session "part" — a session
is the whole PIE run and `recorder.describe_session` returns one flat time
domain. Agents that want to analyse a single race round, the lobby phase
before a race, one sumo round, or one map-editor session must today run a
two-step dance: `recorder.find_events` for the boundary pair, extract the two
anchor timestamps, then feed that window into `get_series` /
`summarize_change` / `get_state` / `query`.

The boundary events already exist in PDS gameplay instrumentation:

- Race round: `race:countdown_start`, `race:run_start`, `race:run_finish`,
  `race:run_abandon`, `race:reset` (`DroneRacingTrack.cpp`)
- Lobby: `mp:session_join`, `lobby:ready_state`, `mp:match_start` /
  `mp:match_end` (`DroneGameMode.cpp`)
- Sumo round: `sumo:round_start` / `sumo:round_end` (`SumoMatchComponent.cpp`)
- Map editor session: `editor:session` with `action: open/exit` and a
  per-session GUID (`MapEditorCharacter.cpp`)
- Missions: `mission:start` / `mission:complete` / `mission:fail` /
  `mission:abandon`; freeflight: `freeflight:start` / `freeflight:end`

Two concrete gaps:

1. **No segment discovery.** Nothing lists "this session contains lobby
   [t0..t1], round 1 [t1..t2], round 2 [t2..t3]". The agent must know the
   event vocabulary per game mode to find the boundaries.
2. **`describe_session` manifest is whole-session only.** The object catalog
   (ranked by activity, capped) and event counts cannot be scoped to a
   window, so in a long multi-round recording the manifest ranking can be
   dominated by an unrelated part of the session and the interesting
   objects for one round may fall below the catalog cap.

**Workaround:** `find_events` → manual window → window-scoped verbs;
`recorder.query` can pair start/end events and bucket series per round in a
Python snippet.

**Fix:** two pieces. (1) New `recorder.list_segments` handler backed by a
declarative pairing-rule table (one row per segment type: start-event
matchers, interior-anchor matchers, end-event matchers, an optional
prop-equality discriminator for single-name pairs like `editor:session`
action=open/exit, an optional correlation prop like `editor_session_id`, and
an open-policy flag). Pairing scans the session's timestamp-sorted event list
once per rule; an end without an open start degrades to a half-open segment
anchored at session MinTs, a start without an end closes at session MaxTs
with `endOpen: true`. Output: `{segments: [{type, index, tMin, tMax,
startOpen?, endOpen?, anchorEvents, props}], meta}` where `props` echoes the
start event's props (run_id, round, editor_session_id, track_uid) and
tMin/tMax feed directly into the existing window-scoped verbs. (2) Optional
`from`/`to` (naming consistent with the other verbs, not tMin/tMax) on
`recorder.describe_session`: when present, the object-catalog activity
ranking is recomputed from change-points inside the window (via the existing
`RecorderAsOf::SliceRange` binary search) and `eventCount` counts only
in-window events; meta echoes the window as timeRange. `race:reset` is an
interior anchor of race rounds, not a boundary (it fires on mid-run soft
resets).

## History
- `#1-initial-report` `OPEN` reporter — Filed after a "can the recorder analyse one race round / lobby / sumo round / editor session?" investigation: possible today only via the find_events→window two-step, with no segment discovery and a whole-session-only describe_session manifest.
- `#2-list-segments-and-windowed-describe` `IN-REVIEW` developer — Implemented both pieces: new `recorder.list_segments` handler (RecorderHandler.cpp) backed by a declarative pairing-rule module RecorderSegments.h/.cpp (seven segment types: race_round, lobby, mp_match, sumo_round, editor_session, mission, freeflight; race:run_start/race:reset are interior anchors; half-open degradation at session edges; editor:session paired via action discriminator + editor_session_id correlation), plus `FRecorderQueryEngine::ListSegments` serialization, and optional `from`/`to` on `recorder.describe_session` that rescopes the activity ranking via `RecorderAsOf::SliceRange` and counts only in-window events. Regression tests added in Tests/Recorder/RecorderSegmentsTests.cpp: FRecorderListSegmentsPairingTest (pairing, anchors, endOpen degradation, reset does not split a round) and FRecorderDescribeWindowedTest (windowed ranking flips to the in-window-busiest object, windowed eventCount, window echoed in meta).
