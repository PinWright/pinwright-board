---
id: E-inspect-object-no-projection-spills
title: "system.inspect.inspect_object has no property projection — inspecting a live PlayerController overflows (91k chars) to read 3 fields, forcing a file dump + grep"
status: WONTFIX
severity: Low
category: ergonomic
tags: [system-inspect, inspect-object, response-size, oversized, projection, readback, docs]
encounters: 3
lastSeen: 2026-07-02T01:24:15.5147610+03:00
---

# `system.inspect.inspect_object` can't scope to a few properties — a normal live UObject read spills to file

`system.inspect.inspect_object` dumps **every** reflected property (+ transform +
components + class) of the target UObject with no way to narrow the result. There is
**no `properties`/`fields` projection and no "just these named props" mode**, so a
perfectly ordinary read of a live object overflows: inspecting the PIE
`PhysicsDemoPlayerController_C_0` produced a **91,718-char** payload (~9x the
10,000-char display threshold), which spilled to
`Saved/EditorAutomation/HttpResponses/.../<uuid>.json`. To confirm the three fields
the task actually wanted — `Pawn`/`AcknowledgedPawn`, `PlayerState`, `Player` — the
agent had to Grep the dumped JSON and issue two Reads (offset 14, offset 1458). One
mandated `inspect_object` became **1 overflowing call + 1 grep + 2 file Reads**.

## Why it matters (process cost in this task)

The task was an end-to-end PIE possession-chain sanity check whose success check
**specifically mandates `inspect_object`** ("inspect_object on that objectPath
succeeds and exposes a possessed-pawn / PlayerState reference"). So the caller cannot
route around it by design — and the natural call is the one that overflows. Friction
note, verbatim:

> "inspect_object's 91k-char response overflowed the 10k display limit and was
> written to a file I had to grep/read."

## What's wrong

`inspect_object` has **zero** size lever. Its own sibling reader in the same
namespace, `system.inspect.list_objects`, already exposes a per-row
`namesOnly`/`fields` projection (`docs/wiki-src/system.inspect.md:38` — valid keys
`label`,`name`,`path`,`class`), so the namespace is internally inconsistent: the list
reader can trim its output but the single-object reader — the one most likely to hit a
property-dense actor like a PlayerController — cannot. This is the same
verbose-reader-with-no-projection-spills shape already accepted for the siblings
`E-niagara-inspect-no-param-readback-projection`,
`E-get-node-details-batch-no-projection-spills`,
`E-inspect-list-objects-no-limit-spills`, and the `*-no-limit-spills` family.

## What it should do

- Add an optional `properties: ["Pawn","PlayerState","Player"]` (and/or a
  `fields`) projection so a scoped read returns only those top-level properties
  inline and stays under the display limit — mirroring `list_objects`' existing
  `namesOnly`/`fields` narrowing.
- **Docs (`docs/wiki-src/system.inspect.md`):** the `inspect_object` section
  cross-references `asset.dump`/`asset.dump_folder` for *content asset* reads
  (lines 9, 24) but never points at `property.get`/`property.list` for reading a
  **single named property off a live UObject** — which is the cheaper, inline path
  today (they exist as targeted single-property getters). Add a "for one or a few
  fields on a live object, prefer `property.get`/`property.list`; a full
  `inspect_object` of a property-dense actor exceeds the inline budget and spills to
  file" pointer until the projection param lands.

## Distinct from

- `E-http-response-spill` (DONE) — the *generic* server-side file-reference spill
  mechanism itself; this ticket is that a *specific* verbose reader (`inspect_object`)
  has no narrowing to stay under the threshold in the first place.
- `F-rpc-property-omit-oversized-opt-in` (DONE) — adds an `omitOversized`
  known-huge-UPROPERTY skip to `property.get`/`property.list`, not `inspect_object`;
  and the PlayerController spill is ordinary whole-object property bulk, not one known
  mega-array.
- `E-inspect-object-class-key-drift` / `B-inspect-object-omits-component-properties` —
  same method, unrelated angles (class-key naming; missing component props). This is
  purely response size / lack of projection.
- `E-inspect-list-objects-no-limit-spills` — identical shape on the *sibling*
  `list_objects` (which nonetheless already has `namesOnly`/`fields`); this is the
  single-object reader that has no projection at all.

severity rationale: impact=pure-friction (response spill forces a Read/grep dance) ×
reach=inspect_object is a common inspector but the overflow only fires on
property-dense objects and `property.get`/`property.list` already offer an inline
single-field workaround (cushioned) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `system.inspect.get_player_controllers` PIE possession-chain task (namespace `system.inspect`, outcome tool_bug for `ui.stop_play` which the judge filed as `B-ui-stop-play-exec-noop`; this is a distinct PROCESS angle on a different method). The mandated `inspect_object` on the live PIE `PhysicsDemoPlayerController_C_0` returned 91,718 chars (~9x the 10k threshold), spilled to `Saved/EditorAutomation/HttpResponses/...json`, and forced 1 overflowing call + 1 Grep + 2 Reads to extract just `Pawn`/`AcknowledgedPawn`, `PlayerState`, `Player`. `inspect_object` has no `properties`/`fields` projection while its same-namespace sibling `list_objects` already exposes `namesOnly`/`fields` (`system.inspect.md:38`). Proposes an optional `properties`/`fields` projection plus a `docs/wiki-src/system.inspect.md` pointer to `property.get`/`property.list` for single-field live reads (the section today cross-refs only `asset.dump` for content assets). Ripgrep across OPEN/closed found no existing ticket on `inspect_object` output spilling or lacking a property projection (`E-http-response-spill` is the generic mechanism; `F-rpc-property-omit-oversized-opt-in` targets property.get/list; `E-inspect-object-class-key-drift` / `B-inspect-object-omits-component-properties` are unrelated angles; `E-inspect-list-objects-no-limit-spills` is the sibling list reader).
- `#2-liveness-single-prop-tags` `OPEN` reporter — Still observed on a realism-mode orientation task (focus `null`, namespace `system`, outcome **done**). To confirm one actor's `Tags` array, `system.inspect.inspect_object {objectPath:"PointLight_1"}` returned `outputTooLong` at **59250 chars** → spilled to a HttpResponses file, forcing a Grep for `Tags` + a line-range Read (offset 1205-1234) to recover `"Tags": {type:TArray<FName>, value:[]}`. Same friction, different object: a single-property check dumps the whole property-dense object because `inspect_object` still has no `properties`/`fields` projection its sibling `list_objects` already exposes. Liveness reconfirm; encounters → 2.
- `#3-liveness-audiocomponent-spill` `OPEN` reporter — Still observed on an audio-attach verify task (focus `audio.play_sound_attached`, namespace `audio`, outcome tool_bug the judge filed as `B-play-sound-attached-not-attached`; this is the distinct response-spill angle). To read back just `AttachParent`/`Sound` on a single `UAudioComponent`, `system.inspect.inspect_object` overflowed at **60,937 chars** → spilled to a `HttpResponses/*.json` file, forcing a Grep + line-range Read to recover the two fields — and it repeated on **all 3** AudioComponent inspections in that one trace (AudioComponent_0, AudioComponent_1, WhooshAudio), ~2 extra tool calls per inspection. Same friction, new object class: still no `properties`/`fields` projection to keep a two-field verify inline. Liveness reconfirm; encounters → 3.
- `#4-wontfix-projection-already-exists` `WONTFIX` developer — The named-property projection this asks to add to `inspect_object` already exists one method over and is the documented path, so adding it would duplicate namespace capability (gold-plating a prominently-recommended general inspector). Verified against synced source, not the lens drafts: (1) `property.list` exposes an exact UPROPERTY allow-list `propertyNames` (`Handlers/Utility/UtilityPropertyHandler.cpp:1623`, genuinely applied :1700-1703) plus a `nameMatch` substring filter (:1622) — a single inline `property.list{objectPath, propertyNames:["Pawn","AcknowledgedPawn","PlayerState","Player"], includeMetadata:false, includeDefault:false, includeOverrideState:false}` returns just the wanted fields, far under the 10k display budget; `property.get` (:1300; `propertyName` supports nested dotted paths :1303, all metadata flags default false :1304-1306) reads a single field inline. The reporter's whole goal (read Pawn/PlayerState/Player, stay inline) is one small `property.list`/`property.get` call today. (2) The docs ALREADY steer live-object reads there: `docs/wiki-src/system.inspect.md:12` routes "Subsystems, GameInstance/GameMode/GameState/PlayerControllers, and live UObject paths" to `runtime-uobject-inspection.md`, which documents `property.get` (:59, worked example :85-92) and `property.list` (:57) for property reads and cross-links `inspect_object`↔`property.get`/`property.list` (:105, :109-110) — that topic page is literally the PIE possession-chain workflow this task was running. (3) The sibling-precedent argument does not transfer: `list_objects`' fields/namesOnly (`EnvironmentHandler.cpp:1387-1388`) and the niagara/get_node_details projections filled REAL gaps (no scoped alternative reader existed), whereas `inspect_object`'s single-object scoped read is already fully served by `property.list`/`property.get`. `inspect_object` is by design a whole-object snapshot; the file spill (`E-http-response-spill`, DONE) is the accepted graceful-degradation path. The success-check "mandate" of `inspect_object` is a fuzz-harness artifact, not a real caller constraint — a real caller reading a few named fields uses `property.list`/`property.get`. Net: a reporter discovery gap (the existing `system.inspect.md:12` → `runtime-uobject-inspection.md` link was not followed), not a code or doc defect. No duplicate, no regression; distinct-angle tickets `E-inspect-object-class-key-drift` / `B-inspect-object-omits-component-properties` unaffected.
