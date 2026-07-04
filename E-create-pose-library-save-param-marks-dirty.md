---
id: E-create-pose-library-save-param-marks-dirty
title: "animation.authoring.create_pose_library `save:true` only marks the package dirty (deferred-save convention) yet echoes existsAfter:true with no persist signal, so the misleadingly-named `save` param forces a follow-up asset.save to actually persist"
status: OPEN
severity: Low
category: ergonomic
tags: [animation, animation-authoring, create-pose-library, save, persistence, mark-dirty, in-memory, naming, no-disk-write]
encounters: 1
lastSeen: 2026-07-05T01:56:03.4709791+03:00
---

# `create_pose_library` `save:true` marks dirty only — the param name implies persistence it does not deliver

`animation.authoring.create_pose_library` takes a `save` param documented as
`save (boolean, optional): Mark asset dirty (default true)`. Despite the name,
`save:true` does NOT write the `UPoseAsset` to disk — it only marks the package
dirty + registers it (the plugin's deliberate deferred-save convention:
`AnimationAuthoringHelpers::SaveAnimAsset` / `McpSafeAssetSave` =
`MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated()`, adopted to avoid
the UE 5.7 bulkdata-corruption-on-immediate-save vector — see the
`E-foliage-add-type-auto-save-undocumented` WONTFIX which documents that helper's
"do not immediately save newly created assets to disk" comment). The create
result still echoes `existsAfter:true` (existence is registry/memory, not disk),
so a caller trusting either `save:true` or `existsAfter:true` believes the pose
library is persisted when it is not — the same latent no-persist shape the
Critical `save-no-disk-write` family confirmed as real asset loss by cold restart.

## Process friction (this task)

REALISM-mode task: stand up a shared pose library for the DinoDragon rig. Clean
10-call trace (wiki discovery, skeleton introspection, create) — no retries, no
errors, no python fallback. The focus verb
`create_pose_library {name:PA_DinoDragon_Poses, path:/Game/Animations, skeleton:SK_DinoDragon_Skeleton}`
succeeded first try (`success:true, existsAfter:true`). But the agent then
reasoned (SAY, verbatim): "The `save` param only marks dirty, so let me persist
it to disk explicitly" and issued a SEPARATE
`asset.save {/Game/Animations/PA_DinoDragon_Poses, force:true}` →
`saved:true, sizeBytes:1718`. So `save:true` on the create call did NOT write the
asset to disk; a second call did. A careful agent recovered with one extra call;
a naive one trusting `save:true` / `existsAfter:true` would have lost the pose
library on the next cold restart or `git reset --hard` — exactly what
`B-audio-create-save-no-disk-write` proved (cold-load-confirmed) for the audio
create verbs that route through the same mark-dirty-only helper shape.

## Why ergonomic here, not a (proven) bug

Unlike the audio / niagara / metasound create verbs — whose `save` param doc does
NOT disclose the mark-dirty-only behavior and which are filed Critical/High for
cold-load-confirmed asset loss — `create_pose_library`'s param IS honestly
documented ("Mark asset dirty"), and this attempt did NOT cold-load-confirm loss
(the follow-up `asset.save` persisted it, 1718 B on disk). So the direct,
evidence-backed defect this run is the misleading SURFACE, not a proven silent
loss:

- The input param name `save` contradicts the plugin's own convention (per
  `B-property-set-saved-true-not-persisted` / the safe-mutation-save contract:
  writers mark dirty; `save` is a separate `asset.save` / `editor.save_all`
  step). A param literally named `save` that does not save is a naming trap.
- The success result echoes `existsAfter:true` with no `saved:false` /
  `pendingFlush` signal to tell the caller a follow-up save is required — so the
  persist requirement is discoverable only by reading the `save` param's fine
  print (or by a failed downstream step).

## Fix (pick one; mirror shipped siblings)

1. Echo a persistence signal in the create result — add `saved:false` + a `note`
   naming `asset.save` / `editor.save_all` as the persist step (matching IN-REVIEW
   `E-level-duplicate-in-memory-only-no-persist-signal`), so the caller learns the
   asset is dirty-only without a failed downstream step; and/or
2. Rename the input param `save` -> `markDirty` (matching the `markedDirty` field
   the IN-REVIEW `B-property-set-saved-true-not-persisted` fix chose for
   cross-handler consistency) so it stops implying disk persistence; and/or
3. If a real disk write on create is wanted, route the `save:true` path through
   the real-save helper the family fixes adopted
   (`SaveAssetToDiskReportingPresence` / `SaveLoadedAssetThrottled` +
   `IFileManager::FileSize` probe) and report an honest disk-gated `saved` —
   `UPoseAsset` is a non-Blueprint/non-SCS asset, so the bulkdata-corruption
   deferral that forced mark-dirty on Blueprint edits does not apply (same
   reasoning as `B-audio-create-save-no-disk-write` #2); and/or
4. Doc note in the `animation.authoring` wiki overlay
   (`docs/wiki-src/animation.authoring.md`) that `create_pose_library` marks dirty
   only and a separate `asset.save` / `editor.save_all` is required before the
   pose library survives restart.

**Workaround:** after `create_pose_library`, call `asset.save {force:true}` (or
`editor.save_all`) before the editor closes or any `git reset --hard` — exactly
what the attempt did (1718 B written).

## Relationship to existing tickets

- `B-create-pose-library-noop-fake-success` (IN-REVIEW) — SAME method, DIFFERENT
  root cause: that ticket is the create-nothing no-op (no `UPoseAsset` was
  produced at all); its fix now constructs + registers a real asset via the
  mark-dirty-only `SaveAnimAsset`, which is precisely the deferred-save behavior
  this ergonomic ticket is about. Not a dup — the naming/persist-signal gap is
  downstream of that fix and untouched by it.
- `B-audio-create-save-no-disk-write` / `B-niagara-save-no-disk-write` /
  `B-metasound-create-save-no-disk-write` (Critical/High, save-no-disk-write
  family) — SAME mark-dirty-only-create root pattern in other namespaces; each is
  method/namespace-specific (own helper), so this is the animation.authoring
  member of that family, filed E- (not B-) because the loss was not cold-load
  confirmed this run and the doc is honest.

severity rationale: impact=misleading-persistence-surface / latent no-persist
mitigated by an honest param doc + one documented recovery call (Low-Medium) x
reach=rare (pose-library authoring is an infrequent path) -> Low — consistent
with the sibling `E-level-duplicate-in-memory-only-no-persist-signal` (Low). The
underlying no-persist is the Critical `B-*-save-no-disk-write` family shape;
flagged here for animation.authoring but not cold-load-confirmed this run.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of a REALISM
  DinoDragon pose-library task (focus `animation.authoring.create_pose_library`;
  10 MCP RPCs, all `ok`, no retries / errors / python fallback — a clean trace).
  Friction: the focus verb
  `create_pose_library {name:PA_DinoDragon_Poses, path:/Game/Animations, skeleton:SK_DinoDragon_Skeleton}`
  returned `success:true, existsAfter:true` with `save:true`, but the asset was
  NOT on disk — the agent (SAY: "The `save` param only marks dirty, so let me
  persist it to disk explicitly") had to issue a separate
  `asset.save {/Game/Animations/PA_DinoDragon_Poses, force:true}` ->
  `saved:true, sizeBytes:1718` to actually persist it. Root cause: the `save`
  param is documented `Mark asset dirty (default true)` and routes through the
  plugin's deferred-save helper (`MarkPackageDirty` + `AssetCreated`, no package
  write), while the result echoes `existsAfter:true` with no `saved:false` /
  `pendingFlush` signal — so the param name `save` implies a persistence the call
  does not perform, and the recovery is discoverable only via the param fine
  print or a failed downstream step. Distinct from the same-method
  `B-create-pose-library-noop-fake-success` (create-nothing no-op, different root
  cause) and is the animation.authoring member of the Critical
  `B-*-save-no-disk-write` mark-dirty-only-create family, filed E- because loss
  was not cold-load-confirmed this run and the param doc is honest. Proposed:
  echo `saved:false` + a persist `note` (per `E-level-duplicate-in-memory-only-no-persist-signal`)
  and/or rename `save` -> `markDirty` (per `B-property-set-saved-true-not-persisted`),
  plus an `animation.authoring` wiki-overlay note. severity rationale:
  impact=misleading-persistence-surface / latent no-persist mitigated by honest
  doc + one documented recovery call (Low-Medium) x reach=rare (pose-library
  authoring) -> Low.
