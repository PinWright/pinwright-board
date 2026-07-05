---
id: E-skeleton-list-bones-no-limit-spills
title: "skeleton.list_bones has no limit/projection/name-filter — a normal 201-bone skeleton overflows the 10k spill threshold and forces a Read of the HttpResponses file just to find one bone"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [skeleton, response-size, oversized, projection, name-filter, docs]
encounters: 3
lastSeen: 2026-06-24T06:10:13Z
---

# skeleton.list_bones has no limit/projection — it dumps every bone and spills to file

`skeleton.list_bones` enumerates **every** bone in a skeleton's reference
skeleton and returns the full array in one payload, with **no `limit`, no name
filter, and no field projection** (SkeletonHandler.cpp:137 registers only the two
resolver params `skeletonPath` / `skeletalMeshPath`). The `RefSkeleton` bones are a
flat, index-ordered array, so — like the shipped sibling `E-actor-list-no-limit-spills`
— the fix is `limit` + projection + a name filter, **not** a pagination cursor. On any
normal humanoid rig the bone count alone pushes the
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

Mirror the narrowing levers already shipped on the canonical sibling
`E-actor-list-no-limit-spills` (`QueryHandler.cpp`) and the same-domain
`E-list-physics-bodies-no-limit-spills` (`PhysicsAssetHandler.cpp`) — **no pagination
cursor** (a flat `RefSkeleton` array does not need one):

- Add an optional `limit` with **default `0` = all** (so the default output stays
  byte-identical for callers that don't pass it) that truncates after filtering while
  `count` reports the returned rows, a new `totalCount` always reports the untruncated
  match count, and a `truncated` flag flips true when rows were elided — the same
  **detectable-elision** contract `E-graph-connections-pagination` and the actor.list /
  physics-bodies siblings landed. **NOT** a "small default that stays inline": bones are
  hierarchy-ordered root-first, so a small default would truncate the *leaf* bones
  (hands/fingers) — exactly the `hand_r` the dominant use case wants.
- Add an optional `nameFilter`/`boneName` case-insensitive **substring** narrowing so the
  "show me every bone matching `hand`" discovery case returns just the matching rows
  inline instead of dumping 201 bones. This is distinct from `skeleton.get_bone_transform`:
  that verb resolves **one exact** bone name (and already returns its `boneIndex` /
  `parentIndex` / `parentName` inline), so it cannot answer "which bones match this
  fragment?" when you don't yet know the exact names — the discovery case
  (`#2`/`#3`, picking source/target bones for `create_virtual_bone`) list_bones is reached
  for. Mirrors the `boneName`/`nameFilter` substring the physics-bodies sibling shipped.
- A `fields` / `namesOnly` projection (via the shared `FHandlerContext::ReadFieldProjection`)
  so the common "just give me the bone names / the hierarchy" case drops the per-bone
  `location` object, which is the bulk of the bytes.
- Reconcile the summary/body mismatch the aside flags: the registration summary claimed
  the method returns the *reference-pose transform* per bone but the body only serialized
  `location` — correct the summary to say *location* (rather than balloon every row with
  rotation/scale, which would only make the spill worse).
- **Docs (`docs/wiki-src/skeleton.md`):** **modify** the existing `### skeleton.list_bones`
  section (it already exists, added by the merged `E-ik-chain-bone-name-discovery`) to note
  that the method returns *all* bones, that the result exceeds the inline budget on any
  normal rig (~200+ bones) and spills, and that `nameFilter` / `limit` / `namesOnly` /
  `fields` keep a listing inline — preserving its existing `add_ik_chain` /
  `get_animation_info` bone-name-source prose.

**Fix:** Add `nameFilter`/`boneName` (case-insensitive substring, snake_case aliases),
`limit` (default `0`=all; emit a new untruncated `totalCount` + a `truncated` flag
alongside the existing `count`), and a `namesOnly`/`fields` per-bone projection (allow-list
of `name`/`index`/`parentIndex`/`parentName`/`location` via
`FHandlerContext::ReadFieldProjection`, defaulting to all so unprojected output is
unchanged) to the `skeleton.list_bones` handler in
`Handlers/Animation/SkeletonHandler.cpp`; correct its summary from "reference-pose
transform" to "reference-pose location"; and modify the `### skeleton.list_bones` section
of `docs/wiki-src/skeleton.md`. No pagination cursor.

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
- `#4-reword-and-implement` `IN-REVIEW` developer — Validity review (correctness/adversarial/historian) confirmed the defect is real and present in current source (SkeletonHandler.cpp:137-188 registered only `skeletonPath`/`skeletalMeshPath`, looped every raw bone :158 emitting name/index/parent + `location` with no cap/filter/projection) and distinct from the sibling family. Reworded three stale framings the lenses flagged, each verified against source: (a) dropped the "no pagination cursor" lever — `RefSkeleton` bones are a flat index-ordered array, and the canonical `E-actor-list-no-limit-spills #4` / `E-list-physics-bodies-no-limit-spills #2` struck the cursor as gold-plating, so this family takes `limit` + projection + name-filter, not a cursor; (b) corrected the `limit` ask from "default small enough to stay inline" (silently-eliding, and *worse* here — bones are root-first so a small default drops the leaf hand/finger bones the use case wants) to default `0`=all with `totalCount`/`truncated` (the detectable-elision contract the family landed, unprojected output byte-identical); (c) corrected the `nameFilter` justification — the ticket claimed `skeleton.get_bone_transform` returns the transform "but not its index/parentIndex," which is FALSE (it returns `boneIndex`/`parentIndex`/`parentName` inline, SkeletonHandler.cpp:262-264). Kept `nameFilter` anyway (against the adversarial "drop it" vote): get_bone_transform resolves one *exact* name, so it cannot serve the multi-match substring *discovery* case (#2/#3, finding source/target bones for `create_virtual_bone`), and the same-domain physics-bodies sibling shipped exactly this substring filter. Implemented the fix: `skeleton.list_bones` now takes `nameFilter`/`boneName` (case-insensitive substring, snake_case aliases), `limit` (default `0`=all; emits the existing `count` plus a new untruncated `totalCount` and a `truncated` flag), and a `namesOnly`/`fields` per-bone projection (allow-list of name/index/parentIndex/parentName/location via `FHandlerContext::ReadFieldProjection`; default keeps all so unprojected output is byte-identical); also corrected the registration summary from "reference-pose transform per bone" to "reference-pose location per bone" (reconciling the aside's summary/body mismatch without ballooning rows). Modified the existing `### skeleton.list_bones` overlay to document the spill + the four narrowing levers + the new `totalCount`/`truncated` returns, preserving its `add_ik_chain`/`get_animation_info` bone-name-source prose (the doc regression tests assert on it). Files: `Source/PinWright/Private/Handlers/Animation/SkeletonHandler.cpp` (handler), `docs/wiki-src/skeleton.md` (docs). Regression test `FSkeletonListBonesNarrowingTest` (`PinWright.skeleton.list_bones.Narrowing`) added to `Source/PinWright/Private/Tests/Gameplay/TestAnimationHandlers.cpp`: builds a real on-disk `USkeleton` with a fixed 5-bone hierarchy (root→spine_01→{hand_r,hand_l,foot_r}; two share the "hand" substring) via `FReferenceSkeletonModifier`, drives the production handler, and asserts `limit=2` caps rows at 2 while `totalCount`=5/`truncated`=true, `namesOnly` drops index/location, and `nameFilter="hand"` returns exactly the 2 matching bones — all against the production handler, so reverting the limit/projection/filter plumbing fails it.
