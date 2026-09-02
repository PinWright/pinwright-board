---
id: B-mrq-artifact-report-omits-stream-count
title: "mrq.run_jobs measures the file it produced but never looks inside it, so an MP4 that carries a 192 kbit/s AAC track for a scene with no audio reports as a clean video-only render"
status: OPEN
severity: Medium
category: bug
tags: [mrq, run_jobs, movie-pipeline, mp4, audio, aac, streams, artifact-report, readback, deliverable, response-honesty, measured]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# The report stats the file; it never opens it

`mrq.run_jobs`' artifact report measures the produced file from the outside — path, existence,
byte size — and derives everything else from numbers the pipeline handed it. It never inspects
the container. So the **number and kind of streams in the deliverable is not a fact any mrq verb
publishes**, and a track nobody asked for rides out of the tool invisibly.

## Measured, 2026-09-02

`ffprobe` on the higher-bitrate flythrough re-render
`Saved/MovieRenders/Atlantis_Flythrough/Atlantis_Flythrough.mp4`
(produced by `mrq.create_job` + `mrq.run_jobs`, 1920x1080, 60 fps, 20.033317 s, 53,650,824 B):

    stream 0  h264  1920x1080  60/1   21,242,124 bps
    stream 1  aac   48000 Hz   stereo    192,000 bps

The container's own tags name the engine's writer, not a post-process:
`stream_tags: encoder=AVC Coding`, `handler_name=SoundHandler`, `major_brand=mp42`.

The audio track is digitally silent for its whole length:

    ffmpeg -i Atlantis_Flythrough.mp4 -map 0:a -af volumedetect -f null -
    n_samples: 1923070
    mean_volume: -91.0 dB
    max_volume:  -91.0 dB
    histogram_91db: 1923070

Every one of 1,923,070 samples sits at the same −91 dB floor — a full-scale-silent stereo track
spending 192 kbit/s. The comparison that makes it a *property of this render path* rather than of
this file: the earlier render of the same sequence,
`Saved/MovieRenders/_prev_flythrough_20260828/LS_Atlantis_Flythrough.mp4`, carries **one stream,
video only** (independently recorded at `Docs/map/atlantis-video-plan.md:48-50`), and the four
ffmpeg-assembled clips from the same session
(`Saved/MovieRenders/Video/{grid/columns,turntable/column,outro/temple_drift}.mp4` and the final
`Atlantis_HowItWasMade.mp4`) are all video-only too. Two deliverables out of one namespace differ
in stream count, and nothing in either response says which is which.

## Mechanism: the report has no container-inspection step, by construction

`BuildArtifactReport` walks the paths MRQ reported and does exactly one file-system operation per
path:

    const FFileStatData Stat = FileManager.GetStatData(*File.Path);
    -- Source/PinWright/Private/Handlers/MRQ/MRQArtifactReport.cpp:144

From that it publishes `exists` (`:146`), `fileSizeBytes` (`:151`), `outputFiles` (`:162`),
`outputFileCount` (`:163`), `measuredFileCount` (`:164`) and `totalFileSizeBytes` (`:167`).
Everything downstream is arithmetic on numbers the *pipeline* supplied, not on the file:
`durationSeconds` is `FrameCount / FrameRate` (`:202-203`), and the header says so in as many
words — *"DURATION IS DERIVED, NOT DEMUXED"* (`MRQArtifactReport.h:45-46`). `overallBitrateBps`
(`:225`) divides the measured byte total by that derived duration, so it is a **whole-file**
figure that silently includes the audio track's 192 kbit/s.

There is no demuxer, no probe, and no stream field anywhere: a case-insensitive grep for
`audio|aac|wave|stream|codec` across all three files of
`Source/PinWright/Private/Handlers/MRQ/` returns only `FEncodeContext` identifiers
(`MRQArtifactReport.cpp:123`, `MRQArtifactReport.h:63,99`, `MRQHandler.cpp:56`) and one prose
comment about the Media Foundation writer (`MRQArtifactReport.h:28`). Nothing reads a track list.

**PinWright does not choose the setting that adds the track**, and that is worth stating so the
fix is aimed correctly. The namespace has exactly one configuration write —
`Job->SetConfiguration(Preset)` (`Source/PinWright/Private/Handlers/MRQ/MRQHandler.cpp:304`) —
which adopts the caller's preset asset wholesale; with no `presetPath` the job keeps the engine's
CDO defaults, and the handler says so itself (`:126-127`: *"the encoder settings are read off it
rather than accepted from the caller, since mrq.create_job takes no encoder params at all"*).
The audio track comes from the engine's MP4 output setting. **The defect is the disclosure**, not
the encode: the plugin published a complete-looking account of a file whose stream list it never
consulted.

## Why it costs something

The deliverable's stated requirement was *"Silent cut. No audio stream at all in the deliverable,
not a silent one"* (`Docs/map/atlantis-video-plan.md` § Production). Because no response field
could answer "does this render have an audio track", the assembly stage had to defend against it
blind — and it had to, because the namespace demonstrably produces both kinds and the caller cannot
tell which they were handed. (The clip that ended up in the cut is the audio-free earlier render;
the AAC-carrying re-render was set aside for an unrelated reason,
`Docs/map/atlantis-video-plan.md:42-57`. The guard is not retrospective luck — it is what a caller
must write when the response will not say.) Two explicit `-an` flags on the encode paths
(`Docs/scripts/video/build_atlantis_video.py:226`, `:692`) plus a standing `ffprobe` gate on the
finished file (`:1160-1170`), whose own docstring records the reason —

    """The plan's decision is a silent cut, which means NO audio stream -- not a silent
    one. Any source clip may carry an audio track, so a stray `-map 0` anywhere would
    smuggle one in; the frame-count gate would not notice, and neither would a viewer
    until upload."""

That is a caller re-implementing, out of band and in another language, the one measurement the
verb that produced the file was best placed to make. The failure mode it guards against is the
same class as the sibling ticket's: every number the render reported stayed correct while the
artifact was wrong.

Second-order, and the reason this is not merely cosmetic: `overallBitrateBps` is the field
`B-mrq-render-result-omits-bitrate-and-size` added as the honest measure of encode quality, and
`bitsPerPixel` (`MRQArtifactReport.cpp:246`) is checked against a plausibility floor. Both are
computed from the **whole file**, so on a file with an audio track they overstate the video
bitrate by the audio bitrate — here 192 kbit/s out of 21.42 Mbit/s, 0.9%, harmless at 1080p60 but
proportionally worse the smaller the video budget. A per-stream breakdown makes both fields exact
instead of approximately right.

## Ask

Publish what the container holds, from the same place the size is measured:

- a `streams` array per output file — `codecType` and `codecName` at minimum, and where cheap the
  per-stream bitrate — so `overallBitrateBps` can be stated beside a `videoBitrateBps`;
- a `warnings` entry when a produced file carries an audio stream and the sequence had no audio,
  which is the case that surprised a caller here;
- OMIT rather than zero when the container could not be read, matching the report's existing
  contract (`MRQArtifactReport.h:31-35`, and `Docs/rpc-design.md` §1/§4).

Implementation note, since the header explicitly declines a heavyweight dependency
(`MRQArtifactReport.h:84-88` reads the encoder knobs by reflection rather than including
`MoviePipelineMP4EncoderOutput.h`, "so this costs no module dependency"): reading an MP4 `moov`/`stsd` track list needs no demuxer and no new module — it is a box
walk over the first megabyte. That keeps the "measured off disk, never inferred" discipline the
report already holds itself to.

## Related

- `B-mrq-render-result-omits-bitrate-and-size` (High, IN-REVIEW) — the same defect class one field
  over: what the render *produced* was not reported. Its landed work added the file-size and
  bitrate measurement this ticket extends inward, from the file's size to the file's contents.
  Filed separately rather than appended as an encounter because that ticket is IN-REVIEW awaiting
  a tester, and a new omission folded into it would blur what is being verified.
- `B-mrq-config-readback-omits-sampling` — the same report's other blind spot, on the input side.
- Lesson: `Plugins/PinWright/Docs/lessons.md:197` records the measurement and the general rule
  (stream count is not evidence of content in either direction; assert silence from measured
  loudness).

## History
- `#1-aac-track-invisible-to-response` `OPEN` reporter — "Filed 2026-09-02 from the Atlantis showcase video. Measured with ffprobe/ffmpeg on the hero MRQ render: stream 1 is `aac` 48 kHz stereo at 192 kbit/s, constant −91.0 dB over 1,923,070 samples, while an earlier render of the same sequence and four ffmpeg-assembled clips from the same session are video-only. The artifact report's only file-system contact is `GetStatData` (`MRQArtifactReport.cpp:144`); duration is derived not demuxed (`MRQArtifactReport.h:45-46`) and no stream/codec/audio field exists anywhere in `Handlers/MRQ/`. PinWright does not choose the encode — its one config write is `SetConfiguration(Preset)` (`MRQHandler.cpp:304`) — so this is a disclosure gap, not an encoder gap. Cost: the assembly stage carries two `-an` flags and a standing ffprobe gate (`Docs/scripts/video/build_atlantis_video.py:226,692,1160-1170`) to re-derive out of band what the verb never said."
