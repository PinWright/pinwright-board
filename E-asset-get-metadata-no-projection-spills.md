---
id: E-asset-get-metadata-no-projection-spills
title: "asset.get_metadata has no field projection — reading a skeleton or anim clip returns ~90k chars and spills to a HttpResponses sidecar, forcing a Bash grep to recover Skeleton/SequenceLength"
status: OPEN
severity: Low
category: ergonomic
tags: [asset, asset-get-metadata, response-size, oversized, projection, docs]
encounters: 1
lastSeen: 2026-07-11T03:23:30.9918175+03:00
---

# `asset.get_metadata` can't scope to a few fields — a skeleton/anim read spills to file

`asset.get_metadata` returns the **entire** metadata blob for the target asset with
no way to narrow the result. There is **no `keys`/`fields` projection and no summary
mode**, so a perfectly ordinary pre-authoring read of a property-dense content asset
overflows the 10k display threshold and spills to a
`Saved/PinWright/HttpResponses/.../<uuid>.json` sidecar. For a `USkeleton` /
`UAnimSequence` the blob carries the full `AnimNotifyList` / `AnimSyncMarkerList` /
`CurveNameList` / `BoneReferences`, so a read of just two-or-three fields returns tens
of thousands of characters.

## Why it matters (process cost in this task)

The task was a Motion Matching setup (`pose_search` namespace) whose pre-authoring
step needed only a handful of fields off Manny's skeleton and clips — the `Skeleton`
binding (to confirm the schema skeleton matches), and `SequenceLength` /
`Number of Frames` for trim planning. The natural read is `asset.get_metadata`, and
it overflowed on **three consecutive** calls:

- `/Game/Characters/Mannequins/Meshes/SK_Mannequin` -> **94,023 chars**
- `/Game/Characters/Mannequins/Animations/Manny/MM_Rifle_Jog_Fwd` -> **86,226 chars**
- `/Game/Characters/Mannequins/Animations/Manny/MM_Run_Fwd` -> **86,244 chars**

All three (~8.6x–9.4x the 10k threshold) spilled to sidecar files, and to recover the
few fields it wanted the agent fell back to **two Bash grep passes** over the dumped
JSON. Friction note, verbatim:

> "asset.get_metadata returning 90k-char payloads spilled to sidecar files, needing a
> grep to pull out clip length/skeleton for trim planning."

Agent's own in-trace line: *"The responses are large. Let me extract the class and
length info from the metadata files."*

The `pose_search` authoring flow itself (`create_schema` -> `create_database` with an
inline animations array -> two `asset.dump` readbacks) was clean and first-try — this
friction is entirely in the generic asset-inspection path used before authoring.

## What's wrong

`asset.get_metadata` has **zero** size lever. A caller who wants one typed field (a
skeleton path, a clip length) must take the whole blob or nothing, then grep the
spill file back out — the exact `verbose-reader-with-no-projection-spills` shape
already accepted for the siblings `E-inspect-object-no-projection-spills`,
`E-blueprint-list-no-projection-spills`, `E-inspect-list-objects-no-limit-spills`, and
the rest of the `*-no-projection-spills` / `*-no-limit-spills` family.

## What it should do

- Add an optional `keys` / `fields` projection (e.g.
  `keys: ["Skeleton","SequenceLength"]`) so a scoped read returns only those
  metadata entries inline and stays under the display threshold — the common
  single-field lookup should never spill.
- **Docs (`docs/wiki-src/asset.md`):** the `asset.get_metadata` section should point
  callers who want a light per-type read (clip length, skeleton binding) at the
  cheaper targeted readers rather than the whole-blob dump — e.g. the animation
  summary readers for `UAnimSequence` — and note that `get_metadata` of a
  notify/curve-dense skeleton or clip exceeds the inline budget and spills to file
  until the projection param lands.

## Distinct from

- `E-http-response-spill` (DONE) / `E-overflow-spill-file-double-encoded` /
  `E-mcp-response-spill-double-escaped` — the *generic* spill mechanism and the
  on-disk spill-file encoding; this ticket is that a *specific* verbose reader
  (`asset.get_metadata`) has no narrowing to stay under the threshold in the first
  place.
- `E-asset-get-doc-promises-tags` — the *opposite* problem on a *different* verb
  (`asset.get` returns too LITTLE — no tags); this is `asset.get_metadata` returning
  too MUCH with no way to trim.
- `E-sequencer-get-metadata-identity-only` — a same-named-but-different-namespace
  method (`sequencer.get_metadata`) that is too THIN; unrelated.

severity rationale: impact=pure-friction (response spill forces a Grep/Read dance) ×
reach=asset.get_metadata is a common inspector but the overflow only fires on
property-dense assets (skeletons, notify/curve-heavy anim clips) and a lighter
identity read (`asset.get`) / type-specific readers exist as a cushion -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a `pose_search` Motion
  Matching setup task (namespace `pose_search`, outcome tool_bug for the
  create/save no-disk-write family which the judge filed as
  `B-pose-search-create-save-no-disk-write`; this is a distinct PROCESS angle on a
  different, generic method). Pre-authoring investigation drove three
  `asset.get_metadata` reads that each overflowed (94,023 / 86,226 / 86,244 chars,
  ~9x the 10k threshold) and spilled to `Saved/PinWright/HttpResponses/*.json`,
  forcing two Bash grep passes to recover `Skeleton` + `SequenceLength` /
  `Number of Frames` for schema-skeleton confirmation and clip trim planning.
  `asset.get_metadata` has no `keys`/`fields` projection or summary mode. Ripgrep
  across OPEN/closed found no existing ticket on `asset.get_metadata` output spilling
  or lacking a projection (`E-http-response-spill` is the generic mechanism;
  `E-asset-get-doc-promises-tags` is the opposite too-thin problem on `asset.get`;
  `E-sequencer-get-metadata-identity-only` is a different-namespace method). Filed
  new within the `projection` family. Proposes a `keys`/`fields` projection plus a
  `docs/wiki-src/asset.md` pointer to lighter type-specific readers.
