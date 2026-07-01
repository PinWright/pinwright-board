---
id: E-skeleton-list-bones-no-limit-spills
title: "skeleton.list_bones has no limit/projection/name-filter — a normal 201-bone skeleton overflows the 10k spill threshold and forces a Read of the HttpResponses file just to find one bone"
status: OPEN
severity: Low
category: ergonomic
tags: [skeleton, response-size, oversized, pagination, projection, docs]
encounters: 3
lastSeen: 2026-06-24T06:10:13Z
---

# skeleton.list_bones has no limit/projection — it dumps every bone and spills to file

`skeleton.list_bones` enumerates **every** bone in a skeleton's reference
skeleton and returns the full array in one payload, with **no `limit`, no
pagination cursor, no name filter, and no field projection**
(SkeletonHandler.cpp:137 registers only the two resolver params `skeletonPath` /
`skeletalMeshPath`). On any normal humanoid rig the bone count alone pushes the
serialized array past the 10000-char spill threshold, so the response comes back
as `outputTooLong` and the full payload is written to
`Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing the caller to
**Read/Grep the spilled file** just to find the index/parent of a single bone
(the overwhelmingly common reason to list bones: "which one is the right hand?").

The friction task's intent at step 2 was tiny — "find the `hand_r` bone in the
list so I can read its transform and anchor a socket to it". One row out of 201.
But the method has no way to ask for "just the bone named `hand_r`", "just the
names", or "just the first N", so the natural "list the bones" call always pays
the full-skeleton dump + spill-to-file + extra Read tax.

## Evidence (replayed verbatim this session)

`call("skeleton.get_info", {skeletalMeshPath:"/Game/Characters/Echo/Meshes/Echo.Echo"})`
→ `{"assetName":"Echo_Skeleton","assetClass":"Skeleton","boneCount":201,...}` — a
perfectly ordinary 201-bone humanoid rig, nothing pathological.

`call("skeleton.list_bones", {skeletalMeshPath:"/Game/Characters/Echo/Meshes/Echo.Echo"})`
→ verbatim:

```
{"outputTooLong":true,"message":"Response exceeds display limit (113391 chars, threshold 10000); full payload written to X:/src/unreal/EAContentExamples57-fuzz1/Saved/EditorAutomation/HttpResponses/20260619T070035Z/20260619T070553Z_<guid>.json","file":{"path":"...json","contentType":"application/json","characters":113391,"threshold":10000}}
```

113391 chars — over **11x** the threshold — for a routine rig. The spill
mechanism itself works correctly (`E-http-response-spill`, DONE); the gap is that
this verbose reader has no narrowing to stay under the threshold in the first
place, so even the simplest "find one bone" call overflows.

## What's wrong

`SkeletonHandler.cpp:137` registers only `skeletonPath` / `skeletalMeshPath`. The
body (`for (int32 i = 0; i < RefSkeleton.GetRawBoneNum(); ++i)`, :158)
unconditionally appends one object per bone — `name`, `index`, `parentIndex`,
`parentName`, and a `location {x,y,z}` from `GetRefBonePose()[i]` (:161-178) —
with no cap, no name filter, and no `fields` gate, then emits the whole `bones`
array (:182). There is no parameter a caller could pass to reduce the payload.

(Aside, separate from this ergonomic gap: the registration summary at
SkeletonHandler.cpp:138 and the generated wiki claim list_bones returns the
"reference-pose transform per bone", but the body only serializes `location`
(:173-176) — rotation and scale from `RefPose` are dropped. Not the friction
here; noted so a fix doesn't re-document a transform it isn't emitting.)

## What it should do

Mirror the fixes already shipped/proposed for the sibling verbose readers:

- Add an optional `limit` (default small enough to stay inline; `0` = all) that
  truncates after iteration while a `count`/`totalCount` keeps reporting the
  untruncated total so elision is detectable — exactly the contract
  `E-recorder-list-sessions-limit` (#2/#4) and `E-graph-connections-pagination`
  (#2/#3) landed.
- Add an optional `nameFilter`/`boneName` substring (or exact) narrowing so the
  dominant "which index is `hand_r`?" case returns the one matching row inline
  instead of dumping 201 bones. (Note `skeleton.get_bone_transform` already gives
  the *transform* of one named bone, but not its `index`/`parentIndex` in a
  whole-skeleton context, which is what list_bones is reached for.)
- Optionally a `fields` / `namesOnly` projection so the common "just give me the
  bone names / the hierarchy" case drops the per-bone `location` object, which is
  the bulk of the bytes.
- **Docs (`docs/wiki-src/skeleton.md`):** note in the `### skeleton.list_bones`
  section that the method returns *all* bones, that the result exceeds the inline
  budget on any normal rig (~200+ bones), and that the proposed
  `limit`/`nameFilter` are the way to keep a listing inline.

## Distinct from

- `E-http-response-spill` (DONE) — that is the *generic* server-side spill
  mechanism (the file-reference fallback itself); this ticket is that a *specific
  verbose reader* has no narrowing to stay under the threshold in the first place,
  the same relationship `E-recorder-list-sessions-limit`,
  `E-graph-connections-pagination`, and `E-volume-get-info-no-limit-spills` have
  to the spill mechanism.
- `E-volume-get-info-no-limit-spills` (OPEN) — identical *shape* (no
  limit/projection → spill → forced Read) but on `volume.get_volumes_info`, a
  different method/handler. Same proposed fix family, different RPC.

## History
- `#1-initial-repro` `OPEN` reporter — Filed from the Echo character-rigging task (focus `skeleton.get_bone_transform`, the finding is about the neighbor `skeleton.list_bones`; outcome ergo). `skeleton.list_bones` has no `limit`/pagination/name-filter/projection (SkeletonHandler.cpp:137 registers only `skeletonPath`/`skeletalMeshPath`; loops all raw bones :158, serializing name/index/parent + a `location` per bone :161-178 with no cap). Replayed live: a perfectly ordinary 201-bone Echo rig (`skeleton.get_info` → `boneCount:201`) produces a **113391-char** `list_bones` payload — 11x the 10000-char spill threshold — returned as `{"outputTooLong":true,...}` and written to `Saved/EditorAutomation/HttpResponses/.../<uuid>.json`, forcing a Read/Grep of the spilled file just to find the one `hand_r` row the task needed. Proposed: add `limit` (default-inline, 0=all, with untruncated `count`) per `E-recorder-list-sessions-limit`/`E-graph-connections-pagination`, a `nameFilter`/`boneName` substring narrowing, optionally a `fields`/`namesOnly` projection, and document the all-bones-overflows-inline behavior in the `### skeleton.list_bones` section of `docs/wiki-src/skeleton.md`. Distinct from `E-http-response-spill` (DONE, the spill mechanism) and `E-volume-get-info-no-limit-spills` (OPEN, the same shape on `volume.get_volumes_info`).
- `#2-second-repro-dinodragon` `OPEN` auditor — Cross-task confirmation on a **different skeleton**, proving the overflow is general (not Echo-specific). Virtual-bone setup task (focus `skeleton.list_virtual_bones`) on `/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon_Skeleton`: step 2's `skeleton.list_bones` again exceeded the 10000-char display limit and auto-paged its full payload to a `Saved/EditorAutomation/HttpResponses/.../*.json` file the agent then had to Read directly to pick valid head/spine/tail/pelvis source/target bone names — exactly the "forced Read tax to find the right bone names" this ticket describes, here for the dominant rigging use case (choosing source/target bones for `create_virtual_bone`). The task's own friction note: "list_bones exceeded the 10k display limit and auto-paged its full payload to a HttpResponses JSON file, which I read directly (expected behavior, not a blocker)" — i.e. the agent normalized the spill+Read as routine, which is precisely the ergonomic cost. Outcome otherwise clean (all 10 calls ok, no is_error). No new proposal; reinforces the existing `limit`/`nameFilter`/`namesOnly` family of fixes.
- `#3-third-repro-dinodragon-skinweights` `OPEN` auditor — Third independent occurrence, again on the DinoDragon mesh (`skeleton.list_bones {skeletalMeshPath:/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon.SK_DinoDragon}`, a 61-bone rig per the task's own `get_info`). Skin-weight cleanup task (focus `skeleton.auto_skin_weights`): step 2's list_bones produced a **34009-char** payload — 3.4x the 10000-char threshold — spilled to `Saved/PinWright/HttpResponses/*.json`, forcing a Read just to scan the 61-bone hierarchy. Confirms the overflow is not limited to big humanoid rigs (201 bones / 113k chars on Echo, #1): even a modest 61-bone creature rig overflows ~3x. The agent again normalized it — friction note verbatim: "list_bones output exceeded the 10000-char display threshold and was spooled to a Saved/PinWright/HttpResponses JSON file, which I read normally (expected behavior, not a real obstacle)" — the third time across these audits an agent reports the spill+forced-Read as routine, which is exactly the ergonomic tax. Otherwise the only friction in an all-`ok` 9-call task; the rest were the user's explicitly-requested step-by-step inspect→prune→auto_skin→normalize→re-verify workflow, not tooling-forced steps. No new proposal; reinforces the `limit`/`nameFilter`/`namesOnly` family. (Note the spill dir is now under `Saved/PinWright/HttpResponses/` here vs `Saved/EditorAutomation/HttpResponses/` in #1/#2 — a path rename, not a behavior change.)
