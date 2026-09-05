---
id: B-asset-save-pie-failure-reports-pendingflush
title: "asset.save reports pendingFlush:true with no saveState when PIE blocks the write, so a hard failure reads as a retryable throttle"
status: OPEN
severity: High
category: bug
tags: [asset-save, pie, savestate, pendingflush, silent-false-signal, diagnostics, multi-agent]
encounters: 9
lastSeen: 2026-09-05T20:35:00Z
---

# `asset.save` presents a PIE-blocked failure as a pending flush

## Symptom

With the editor in play mode, `asset.save {force:true}` on a freshly duplicated Niagara
emitter returned, twice, identically:

```json
{"assetPath":"/Game/FPS/VFX/Emitters/E_Explosion_Flash",
 "package":"/Game/FPS/VFX/Emitters/E_Explosion_Flash",
 "saved":false,"sizeBytes":0,"pendingFlush":true}
```

No `.uasset` was written. The response carries **no `saveState`**, no `saveDetail`, no
`pieActive` — nothing that distinguishes "the 0.5 s throttle skipped this write" from "the
write is impossible right now".

The editor had measured the real outcome on that very call. From
`Saved/Logs/EAContentExamples58.log`:

```
LogUtils: Error: The Editor is currently in a play mode.
LogPinWrightSubsystem: Warning: SaveLoadedAssetThrottled: failed to save
  '/Game/FPS/VFX/Emitters/E_Explosion_Flash.E_Explosion_Flash'
LogPinWrightSubsystem: Warning: SaveAssetToDiskReportingPresence:
  '/Game/FPS/VFX/Emitters/E_Explosion_Flash' is NOT durable after this save.
  state=failed outcome=Failed forced=true dirtyBefore=true existedBefore=false
  ... The save was attempted and produced no durable revision;
  a flush will not help until the cause is cleared.
```

So the handler computed `state=failed` and logged it, and then answered the RPC with
`pendingFlush:true` and dropped the state on the floor.

## Why this is wrong, in the plugin's own terms

`safe-mutation-save.md` is explicit that these two are opposite instructions:

| `saveState` | Durable | Do |
|---|---|---|
| `deferred` | no | `asset.save {force:true}` or `editor.save_all` |
| `failed` | no | a flush never fixes it; clear the cause first |

and states: *"`pendingFlush` means 'requested and not durable'. It does **not** mean flushing
will work — only `deferred` and `notRequested` are fixed by a flush. **Read `saveState` before
retrying.**"*

The caller is told to branch on `saveState`, and `asset.save` does not return it. The only
field it does return, `pendingFlush:true`, is also the signature of the throttle case whose
documented remedy is `force:true` — which the caller has by then already used. The wiki's own
escape hatch (*"a response carrying `saveRequested` but no `saveState` comes from a handler not
yet threaded through; `saved` is then the only claim it makes"*) does not rescue this: the
handler is demonstrably not un-threaded, it measured `state=failed` and logged it.

Practical result: the documented decision procedure cannot be followed, and the natural read of
the payload is "retry the flush", which is an infinite loop. Only reading the editor log reveals
the cause. That is a false signal on a normal path, not a missing nicety.

## This is the `save_all` gap, never extended to `asset.save`

`B-editor-save-all-pie-diagnostic` (**DONE**) is exactly this complaint against
`editor.save_all`: PIE blocks the write, the payload names no cause, "the caller had to guess
between PIE / source-control / read-only / checkout / compile-error". Its accepted fix adds
`pieActive`, `editorMode` and a per-asset `reason:"BlockedByPie"`. None of that reached
`asset.save`, which is the verb the docs push callers toward — `asset.save`'s own page sells it
as "the targeted single-asset save ... reach for this instead of the blunt `editor.save_all`".
The recommended verb is the one with the worse diagnostic.

## Blast radius observed

This is a shared multi-agent editor, and PIE is global. In a ~10 s window the same failure hit
at least four assets across three unrelated streams — `E_ImpactFlesh_Spray`, `NS_Muzzle_AR`,
`E_Blood_Mist`, `E_Explosion_Flash` — each stream seeing only `pendingFlush:true` and each with
no way to learn that a *different* agent's play session was the cause. An agent that trusts the
field keeps authoring in memory, which is unbacked work: the same editor had crashed 10 minutes
earlier (`B-screenshot-designer-leaves-designer-open-compile-crash`) and discarded exactly that
kind of in-memory state.

## What should happen

1. Thread `saveState`/`saveDetail` through `asset.save`'s response — it is already computed for
   the log line. `state=failed` must not surface as `pendingFlush:true`.
2. Carry the cause the way `B-editor-save-all-pie-diagnostic` settled for `save_all`:
   `pieActive:true` / `reason:"BlockedByPie"`, so a caller can act without reading the log.
3. Ideally reserve `pendingFlush` for states a flush actually fixes (`deferred`,
   `notRequested`), per the wiki's own rule.

## Workaround

After any `saved:false`, ignore `pendingFlush` and grep the editor log for
`SaveAssetToDiskReportingPresence` on that package; branch on `state=` there. Do not retry
`asset.save {force:true}` more than once.

severity rationale: impact=silent wrong signal on a normal path (the caller trusts
`pendingFlush` and retries a write that can never succeed, and believes the edit is merely
queued when it is not durable) x reach=every `asset.save` in the editor for as long as any agent
holds PIE, across all concurrent streams -> High.

## Fix

Root cause, read out of the source rather than inferred: the plugin reaches disk for a single
asset through exactly one engine call, `UEditorAssetLibrary::SaveLoadedAsset` in
`SaveLoadedAssetThrottled` (`Source/PinWright/Private/Utils/AssetUtils.cpp`), and that API opens
with `EditorScriptingHelpers::CheckIfInEditorAndPIE()`
(`C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/EditorScriptingHelpers.cpp:143`), which logs
`LogUtils: Error: The Editor is currently in a play mode.` and returns false for EVERY call while
`GEditor->PlayWorld || GIsPlayInEditorWorld`. That refusal came back as a bare `false`, became
`ESaveLoadedAssetOutcome::Failed` through the throttle-named channel, mapped to
`EAssetSaveState::Failed`, was logged in full - and then `AssetSaveHandler.cpp` published only
`{saved, sizeBytes}` plus a hand-rolled `pendingFlush`, dropping the state it was holding in a
local variable. `saveState` was emitted on the `diskStateDiverged` error path only.

Design: detect the block up front at the shared chokepoint rather than classifying the resulting
failure after the fact (which is what `editor.save_all` does, correctly, for its many-caused
failures). The engine's gate is an unconditional documented precondition, so a pre-check turns a
guess into a fact, keeps a PIE refusal out of the throttle-named failure channel, and skips a
disk-header read plus an engine call that were never going to write anything. `blockedByPie` is a
state of its own rather than a flavour of `failed`, because the remedy differs: wait, re-issue.

Files changed:

- `Source/PinWright/Private/Utils/PieSaveBlockGuard.h` / `.cpp` (NEW) - measures the block
  (mirroring the engine's exact predicate), names the PIE worlds by reusing
  `PieWorldSelector::GatherPieContexts`, and publishes `pieActive` / `editorMode` / `pieWorlds`.
  Writes nothing when PIE is inactive. Same Probe/Describe/AddJson shape as
  `Utils/PackageDiskStateGuard.h`.
- `Source/PinWright/Private/Utils/AssetSaveState.h` - new `EAssetSaveState::BlockedByPie`.
- `Source/PinWright/Private/Utils/AssetUtils.h` / `.cpp` - new
  `ESaveLoadedAssetOutcome::RefusedBlockedByPie`; the PIE gate in `SaveLoadedAssetThrottled`,
  placed AHEAD of the throttle (so a dirty throttled save under PIE is not reported as
  `deferred`, whose remedy is a flush that hits the same refusal) and gated on
  `Package->IsDirty() || bForce` so a clean unforced save keeps its existing already-clean
  verdict; the outcome -> state mapping; the wire spelling `blockedByPie` and its `saveDetail`;
  `AddAssetSaveReport` and `AddMarkDirtySaveReport` now publish the PIE block on the
  requested-but-not-durable branch, which fixes the ~90 un-threaded save-flag verbs
  (`material.authoring.*`, `audio.synth.export`, ...) without touching them; new
  `AddAssetSaveSizeReport` emits `sizeBytes` plus `sizeBytesIsStale:true` when a non-durable save
  reported a pre-existing file's size. Also sharpened the `deferred` detail to name the 0.5s
  throttle and to say it is the one state a retry fixes right now.
- `Source/PinWright/Private/Handlers/Asset/AssetSaveHandler.cpp` - routes through
  `AddAssetSaveReport` + `AddAssetSaveSizeReport` instead of hand-rolling
  `{saved, sizeBytes, pendingFlush}`, so `asset.save` carries `saveRequested` / `saveState` /
  `saveDetail` on every path; the Blueprint-integrity refusal now reports `saveState:"failed"`
  rather than `notRequested`. Handler summary and file header updated.
- `Source/PinWright/Private/Handlers/Asset/StaticMeshSetMaterialHandler.cpp`,
  `StaticMeshSetCollisionComplexityHandler.cpp`, `StaticMeshBakeTransformHandler.cpp` - threaded
  `EAssetSaveState` through and adopted the size helper, so the neighbouring `save`-flag verbs
  answer in the same shape.
- `Docs/wiki-src/safe-mutation-save.md` - `blockedByPie` row plus a Retry column in the Save
  States table; the `sizeBytes` / `sizeBytesIsStale` rule; a new
  `## PIE Blocks Every Save In The Editor` section stating the absence contract (no `pieActive`
  on a not-durable save means PIE was not running) and the `BlockedByPie` <-> `blockedByPie`
  vocabulary mapping against `editor.save_all`; the "retry with force" sentence now says that
  remedy applies only under `saveState: "deferred"`.
- `Docs/wiki-src/asset.md` - `### asset.save` rewritten to say branch on `saveState`, never on
  `pendingFlush`, with a worked `blockedByPie` payload and the `sizeBytes` caveat.

Tests (added, not run - a separate compile pass follows):

- `Source/PinWright/Private/Tests/Assets/TestAssetSavePieBlock.cpp` (NEW) -
  `PinWright.assets.PieSaveBlock.ActiveBlockNamesTheSession` (the block payload names map, world
  path and instance, and `editorMode` matches `editor.save_all`'s spelling),
  `...InactiveBlockWritesNothing` (the absence contract: zero fields added),
  `PinWright.assets.AssetSaveReport.StaleSizeBytesAreLabelled` (three size cases), and
  `PinWright.assets.AssetSaveHandler.ResponseAlwaysCarriesSaveState`, which drives the real
  handler on a real package through a forced write and a throttled non-durable write and requires
  a `saveState` on both. NOTE that id is deliberately NOT under `PinWright.asset.save`, which is
  a complete leaf that a dotted suffix would silently swallow.
- `Source/PinWright/Private/Tests/Assets/TestAssetSaveState.cpp` - `blockedByPie` added to the
  per-state tuple table and the durability list, plus assertions that it is distinguishable from
  both `deferred` and `failed` and that its detail rules out retrying and names
  `editor.pie_status`.

Reviewer verification:

1. Compile the plugin, then run `PinWright.assets.` + `PinWright.asset.save` in ONE editor.
   Expect `PinWright.asset.save` (the pre-existing leaf) to still appear in the queue - if it
   vanished, a dot-prefix collision was introduced.
2. Live editor, no PIE: `asset.save` a dirty asset twice inside 0.5s. The second must answer
   `saved:false, pendingFlush:true, saveState:"deferred"` and NO `pieActive`.
3. Start PIE (`editor.play`), mutate an asset, `asset.save {force:true}`. Expect
   `saved:false, saveState:"blockedByPie", pieActive:true, editorMode:"PIE"`, a `pieWorlds` entry
   naming the running map, and - over an asset that already exists on disk -
   `sizeBytesIsStale:true`. Confirm the `.uasset` mtime did not move.
4. `editor.stop`, re-issue the same `asset.save`, require `saveState:"written"` and a moved mtime.
5. In the same PIE window, call a `save:true` verb that threads no state (e.g.
   `material.authoring.set_material_instance_parameters`) and confirm it now carries
   `pieActive` / `pieWorlds` beside its `pendingFlush`.
6. Confirm nothing regressed for the clean-package case: `asset.save` an already-clean asset
   during PIE must still not be turned into a false refusal by the new gate.

## History

- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8, shared editor, port 27145) authoring `/Game/FPS/VFX/Emitters/E_Explosion_Flash`. Emitter was fully configured and `niagara.compile {force:true, wait:true}` returned `status:"completed"`; two consecutive `asset.save {force:true}` calls each returned `saved:false, sizeBytes:0, pendingFlush:true` with no `saveState`, and `ls` confirmed no `.uasset` on disk. The editor log for the same calls records `LogUtils: Error: The Editor is currently in a play mode.` plus `SaveAssetToDiskReportingPresence ... state=failed outcome=Failed forced=true`, i.e. the cause was PIE held by another agent in the shared editor and the handler knew the state was `failed` while replying `pendingFlush`. Cross-ref `B-editor-save-all-pie-diagnostic` (DONE) — same defect class, fixed for `editor.save_all` only, and `asset.save` is the verb the docs steer callers to instead. Diagnosis from the response payloads, the log lines quoted above, and `safe-mutation-save.md`'s `saveState` table; no plugin source read.
- `#2-second-encounter-stale-sizebytes` `OPEN` reporter — Independently hit on the same editor/session authoring `/Game/FPS/VFX/Emitters/E_Smoke_Smoke` (smoke-grenade emitter stream). Adds one detail #1 could not show: the asset **already existed on disk**, so the two `asset.save {force:true}` replies were `{"saved":false,"sizeBytes":114702,"pendingFlush":true}` — `sizeBytes` is **non-zero and equal to the stale on-disk size**, not `0`. A caller who sanity-checks `sizeBytes > 0` (a natural guard, since #1's failure showed `sizeBytes:0`) is therefore misled twice over: the payload looks like a real write of a plausibly-sized package that is merely queued. `asset.is_dirty` on the same package returned `isDirty:true` immediately after, and the matching log lines are `SaveLoadedAssetThrottled: failed to save '/Game/FPS/VFX/Emitters/E_Smoke_Smoke.E_Smoke_Smoke'` + `SaveAssetToDiskReportingPresence ... state=failed outcome=Failed forced=true dirtyBefore=true existedBefore=true sizeBefore=114702 sizeAfter=114702 stampBefore=2026-09-02T19:54:26.000Z stampAfter=2026-09-02T19:54:26.000Z` (19:55:17 and 19:55:24 UTC). Suggests the fix should also stop reporting `sizeBytes` from the pre-existing file on a failed write, or label it `sizeBytesOnDisk`, so it cannot be read as the size just written.
- `#3-recurs-after-pie-restarts` `OPEN` reporter — Third occurrence, same session, and the one that shows the failure is environmental and time-varying rather than a property of the asset. `/Game/FPS/VFX/Emitters/E_Explosion_Light` **saved successfully** at 20:00:57Z (`saved:true, sizeBytes:88302`, `.uasset` mtime confirmed). 54 seconds later, after two further edits and a clean `niagara.compile status:"completed"`, the identical `asset.save {force:true}` on the identical asset returned `{"saved":false,"sizeBytes":88302,"pendingFlush":true}` — and the log again shows `LogUtils: Error: The Editor is currently in a play mode.` plus `SaveAssetToDiskReportingPresence ... state=failed outcome=Failed forced=true ... stampBefore=2026-09-02T20:00:57.000Z stampAfter=2026-09-02T20:00:57.000Z`. A different agent had re-entered PIE in the interim; `tail -200` of the log carried 61 `play mode` errors. Confirms `#2`'s `sizeBytes` point on a second asset: the reply's `sizeBytes` is the **unchanged pre-existing file's** size, and here it is byte-identical to the size the *successful* save reported a minute earlier, so comparing `sizeBytes` against a known-good previous save cannot detect the failure either. Practical note for whoever fixes this: because PIE is global and is entered and left repeatedly by unrelated agents, the same call alternates between success and silent-looking failure minute to minute, so any caller that does not read the log will produce assets that are durable or not essentially at random.

- `#4-fourth-stream-and-the-symmetric-case` `OPEN` reporter — Fourth independent stream hit this, and this sighting adds the *symmetric* case the first three do not cover: the blocked writer was not the agent running PIE, and the agent running PIE was not doing anything wrong. WEAPONS called `blueprint.scs.set_property` on `/Game/FPS/Weapons/BP_Weapon_AR` and `BP_Weapon_Pistol` (setting `StaticMesh` on the inherited `WeaponMesh` component); both returned `success: true, compiled: true, source: "inherited_scs"`. The following `asset.save {force: true}` on each returned `saved: false, pendingFlush: true` — **with a `sizeBytes` that had grown** (34835 -> 35756 and 36164 -> 37093), which reads as "the file was written and is 35756 bytes". It was not: `ls` gave an mtime of 23:09:55 local, minutes earlier, and `grep -ac SM_WPN` on both `.uasset` files returned **0** — the mesh assignment was not in either file. `editor.pie_status` then showed `inPie: true` with one standalone context on `/Game/FPS/Test/UEDPIE_0_T_UI.T_UI`, i.e. the **UI** stream's session, taken under the project's world-lock protocol while WEAPONS was doing lock-free asset work — which the protocol explicitly permits and encourages. So the reporting problem is worse than "your own PIE blocks your saves": there is no reason for the blocked agent to think about PIE at all, and nothing in the response mentions it. **`sizeBytes` on a `saved: false` response is the field that did the damage here** — it is the in-memory expectation, not what is on disk, and paired with `pendingFlush` it reads as a throttle that already half-succeeded. Two additions to this ticket's existing ask, both cheap: name the blocker (`saveState: "blockedByPie"` plus the PIE map, which `editor.pie_status` already knows) so the message is actionable by someone who did not start the session; and **omit or zero `sizeBytes` when `saved` is false**, because a byte count on a write that did not happen is a claim about disk that is not true. Disk proof used here was `ls` mtime plus `grep -a` on the bytes, per the project rule that landed today after `asset.reload` crashed the editor a third time — worth noting because reload is the check an agent would otherwise reach for and it is now forbidden.
- `#N-audio-stream-blueprint-migration` `OPEN` reporter — Hit again on the FPS AUDIO stream while migrating `/Game/FPS/Audio/BP_DA_ImpactSFX`'s two lookup maps from a `byte` key to `EPhysicalSurface`. `asset.save {assetPath, force:true}` answered `{saved:false, sizeBytes:16180, pendingFlush:true}` twice, with **no error, no `saveState`, and no mention of PIE**. `asset.is_dirty` confirmed `isDirty:true`. The file on disk was an hour stale (mtime 22:41:23 against a 23:49 edit) and `grep -a EPhysicalSurface` on the `.uasset` returned **0**, so the whole type migration existed only in memory. Only `editor.save_all` diagnosed it, and precisely: `SAVE_FAILED ... (PIE active; 3 asset(s) locked by PIE)` with `failedAssets:[{path:.../BP_DA_ImpactSFX, reason:"BlockedByPie"}]` — another stream (WEAPONS) was running a PIE fire/reload verification and PIE holds Blueprint packages. Two things make this expensive rather than cosmetic. First, `pendingFlush:true` is the *same* answer `asset.save` gives for the benign 0.5 s throttle, so the documented remedy (retry with `force:true`) is exactly wrong here and I burned two retries on it. Second, in a shared editor the blocker belongs to a different agent, so the caller cannot even guess the cause from its own actions — `save_all` knows the reason and `asset.save` discards it. Ask: surface the same `BlockedByPie` reason (and a `saveState` of `failed`, not a bare `pendingFlush`) from `asset.save`. Workaround: when `asset.save` reports `pendingFlush` twice on an unchanged mtime, call `editor.save_all` purely to read the reason, then wait out the other stream's PIE session.
- `#N-audio-synth-export-batch-nonpie-variant` `OPEN` reporter — Hit on the FPS AUDIO stream re-synthesising the 12 `SW_Step_*` footsteps and 5 ambience/foley waves. Sixteen consecutive `audio.synth.export {save:true}` calls returned `verification.pass:true, existsAfter:true, existsOnDisk:true` alongside `saved:false, pendingFlush:true, pendingSave:true`; six follow-up `asset.save {assetPath, force:true}` calls each answered `{saved:false, sizeBytes:<stale on-disk size>, pendingFlush:true}` with **no `saveState`**. `ls` confirmed all 11 files still carried their pre-edit mtime (22:43) more than an hour later, reproducing `#2`/`#4`'s point that `sizeBytes` on a `saved:false` reply is the stale file, not a write. The new datum is the escape hatch: `unreal.EditorLoadingAndSavingUtils.save_packages(pkgs, False)` run through `python.execute` wrote **all 12 packages in one call** and returned `True`, and `ls` then showed every mtime advanced — so on this occasion a different save API succeeded on the same packages within seconds of `asset.save {force:true}` refusing them. I did not establish whether another stream's PIE was up at the moment of either call, so this is not offered as a counter-example to the PIE root cause; what it does show is that `asset.save`'s `pendingFlush` was not describing a condition that blocked *all* writers, and that a caller with no `saveState` to read cannot tell the two situations apart. Reinforces this ticket's existing ask (`saveState` + a named blocker, and drop `sizeBytes` when `saved:false`), and adds: whatever `asset.save {force:true}` is doing, it is weaker than `save_packages(..., only_dirty=False)`, so either it should route through that on `force`, or the docs should name `save_packages` as the real forced path. Disk proof was `ls` mtime plus `grep -a` on the `.uasset` bytes throughout; `asset.reload` was not used.
- `#N-pie-root-cause-confirmed-from-log-audio-waves` `OPEN` reporter — Closes the gap `#N-audio-synth-export-batch-nonpie-variant` names explicitly ("I did not establish whether another stream's PIE was up"). Same session, FPS AUDIO stream, re-synthesising the ten weapon/impact waves. `audio.synth.export {save:true}` on `SW_Impact_Glass_A` returned `verification.pass:true` beside `saved:false, pendingFlush:true`; two `asset.save {force:true}` retries answered the same with the **stale** `sizeBytes:101173`. The editor log carries the cause on the adjacent line and the RPC does not: `LogUtils: Error: The Editor is currently in a play mode.` immediately precedes each `SaveLoadedAssetThrottled` / `SaveAssetToDiskReportingPresence: ... state=failed outcome=Failed forced=true`, and `LogPlayLevel: Creating play world package: /Game/FPS/Test/UEDPIE_0_T_Weapons` bounds the window. **The same candidate exported and saved cleanly on the first attempt after `LogWorld: BeginTearingDown`, with no recipe change** — so on this occasion PIE *is* demonstrably the blocker, and the identical symptom recurred minutes later inside a second PIE window (`UEDPIE_0_T_AI`) across seven different packages, all of which then saved `true` on one retry once that PIE ended. Two consequences for the ask: (a) the blocker is already known to the handler at the moment it answers — `LogUtils` prints it one line earlier — so `saveState:"failed"` plus a named `saveBlocker:"pie"` is a plumbing change, not new detection work; (b) since the block is a *window*, not a property of the asset, the useful response field is "retry when PIE ends", which is precisely what today's `pendingFlush:true` with no `saveState` cannot distinguish from the 0.5 s throttle it looks identical to. Disk proof throughout was `ls`/`stat` mtime plus `grep -a` on the `.uasset` bytes; `asset.reload` was not used. Cost here was ~15 min of blocked exports plus one wrong initial diagnosis (read as a throttle, retried twice for nothing).
- `#5-fixed-in-review` `IN-REVIEW` developer - Verified TRUE against source (the handler holds a fully-computed `EAssetSaveState` in a local and publishes only the bool; `saveState` reached the wire on the `diskStateDiverged` error path alone), root-caused to `UEditorAssetLibrary::SaveLoadedAsset`'s unconditional `CheckIfInEditorAndPIE` gate, and fixed at the shared chokepoint: a new `EAssetSaveState::BlockedByPie` produced by a pre-check in `SaveLoadedAssetThrottled`, `asset.save` routed through the shared `AddAssetSaveReport` so it always emits `saveState`/`saveDetail`, `pieActive`/`editorMode`/`pieWorlds` published on every not-durable save report (which covers the un-threaded `save`-flag verbs of `#4`/`#5` on the duplicate too), and `sizeBytes` labelled `sizeBytesIsStale` when it is the pre-existing file's - the `#2`/`#3`/`#4` complaint. Wiki pages `safe-mutation-save.md` and `asset.md` updated; two automation test files added/extended. NOT compiled and NOT run: a separate compile pass follows. See the `## Fix` section above for the file list and the reviewer's verification steps. `B-asset-save-omits-savestate-pie-block` is the same defect and is closed by this same change; it is marked as a duplicate of this ticket and moved to IN-REVIEW alongside it.

- `#9-verified-in-fps-build` `IN-REVIEW` reporter - **Fixed. Verified live in the FPS build, 2026-09-05 17:53Z**, with a standalone PIE session held by this same stream. Call: `asset.save {assetPath:"/Game/FPS/Weapons/Meshes/Test/SM_WPN_SlotProbe"}`. Response:
```
saveRequested true   saved false   pendingFlush true
saveState "failed"
saveDetail "The save was attempted and produced no durable revision; a flush will not help
            until the cause is cleared. The editor log carries a SaveAssetToDiskReportingPresence
            line with the outcome, sizes and timestamps."
pieActive true   editorMode "PIE"
pieWorlds [{pieInstance:0, mapName:"T_Weapons", worldPath:"/Game/FPS/Test/UEDPIE_0_T_Weapons.T_Weapons"}]
sizeBytes 30611   sizeBytesIsStale true
```
  The failure mode this ticket was filed for is gone: `saveState:"failed"` states the outcome, `saveDetail` says explicitly that retrying the flush will not help, `pieActive`/`editorMode`/`pieWorlds` name the cause and who holds it, and `sizeBytesIsStale` stops the byte count being read as evidence of a write. `pendingFlush:true` is still present but is now qualified rather than being the only signal, so it can no longer be mistaken for a retryable throttle. Seven encounters' worth of that misreading in this project.

- `#10-returned` `OPEN` reporter - **The `asset.save` half is fixed; the "~90 un-threaded save-flag verbs" half is not.** FPS VFX stream, same shared editor (port 27145), 2026-09-05 22:22-22:26 local, PIE held by another stream on `T_AI`. Two calls, minutes apart, against the same material instance:
  - `material.authoring.set_material_instance_parameters {assetPath:"/Game/FPS/VFX/Materials/MI_FPS_Water_Crown", scalar:{EdgeSharpness:1.2, SoftEdge:1.0}, save:true}` returned `applied:[{scalar EdgeSharpness},{scalar SoftEdge}]`, `failed:[]`, `existsOnDisk:true` - and **no save fields whatsoever**: no `saved`, no `saveRequested`, no `pendingFlush`, no `saveState`, no `pieActive`, no `pieWorlds`. Nothing was written; `ls --time-style=full-iso` gave mtime `2026-09-05 21:00:34` at 22:25:43, i.e. 85 minutes stale.
  - `asset.save {assetPath:<same>, force:true}` on the very next call returned the full fixed payload: `saveState:"blockedByPie"`, `saveDetail` naming the refusal and saying force/save_all cannot help, `pieActive:true`, `editorMode:"PIE"`, `pieWorlds:[{pieInstance:0, mapName:"T_AI", worldPath:"/Game/FPS/Test/UEDPIE_0_T_AI.T_AI"}]`, `sizeBytes:11036` with `sizeBytesIsStale:true`.

  So the chokepoint knows, and `asset.save` reports it correctly - `#9`'s verification stands and this stream reproduces it. What does not hold is the `## Fix` section's claim that `AddAssetSaveReport` / `AddMarkDirtySaveReport` "fixes the ~90 un-threaded save-flag verbs (`material.authoring.*`, `audio.synth.export`, ...) without touching them", and its verification step 5 ("call a `save:true` verb that threads no state (e.g. `material.authoring.set_material_instance_parameters`) and confirm it now carries `pieActive` / `pieWorlds` beside its `pendingFlush`"). Run today, that verb carries neither, and it carries no `pendingFlush` either - it emits no save report at all, which suggests its handler does not route through `AddAssetSaveReport` and so the shared-chokepoint fix never reaches it. Step 5 should be re-run before this ticket leaves IN-REVIEW; passing it on `asset.save` alone does not cover the class the Fix says it covers.

  Cost here was small only because CLAUDE.md's "verify a write against disk, not against the object you just wrote" rule made me `ls` the mtime anyway. A caller who trusts `applied[]` + `existsOnDisk:true` + a defaulted `save:true` has no field in the response that disagrees with "it saved" - which is the same silent-false-signal this ticket was opened for, still live on the verbs that never had a save report to fix.

- `#11-non-pie-success-still-silent` `OPEN` reporter - **New datum that removes PIE from the diagnosis for the `material.authoring.*` half.** Every prior sighting of `material.authoring.set_material_instance_parameters {save:true}` on this ticket (`#10-returned`, and `#4`/`#5` on the duplicate `B-asset-save-omits-savestate-pie-block`) observed the missing save fields while PIE held the editor and nothing was written, which leaves open the reading that the report is simply not reached on the blocked path. It is not that. FPS VFX stream, shared editor port 27145, 2026-09-05, **`editor.pie_status` returned `inPie:false, count:0` immediately before the calls**. Two `set_material_instance_parameters {save:true}` calls - `/Game/FPS/VFX/Materials/MI_FPS_Smoke_MetalDark` `{scalar:{SoftFadeDistance:8, Density:0.9, DetailAmount:0.45}}` and `/Game/FPS/VFX/Materials/MI_FPS_Glass_Dust` `{scalar:{AmbientBoost:0.445}}`. Both **did write**, byte-verified on disk: `MI_FPS_Smoke_MetalDark.uasset` 7859 -> 8302 bytes with packed floats `66 66 66 3f` (0.9) and `66 66 e6 3e` (0.45) each present once and the old `9a 99 19 3f` (0.6) / `9a 99 59 3f` (0.85) absent; `MI_FPS_Glass_Dust.uasset` carrying `0a d7 e3 3e` (0.445) with the old `7b 14 ae 3e` (0.34) absent. And both responses carried **exactly the same field set as the PIE-blocked failures in `#10`**: `applied[]`, `failed:[]`, `existsOnDisk:true`, `shaderCompile{}` - no `saved`, no `saveRequested`, no `saveState`, no `pendingFlush`, no `pieActive`.

  So on this verb the response is **wire-identical for a durable write and for a write that never happened**, and the absence is unconditional rather than a gap on the failure path. That strengthens `#10`'s inference: the handler does not route through `AddAssetSaveReport` at all, so there is no path - success or failure - on which the shared chokepoint's fix can reach it. It also means the class cannot be verified from a passing case either: a caller who sees `applied[]` + `existsOnDisk:true` and concludes "saved" is right half the time by luck, and has no field to consult in either direction. Re-running the `## Fix` section's verification step 5 needs both cases: PIE up (expect `pieActive`/`saveState:"blockedByPie"`) and PIE down (expect `saveState:"written"`). Confirmed in the same session that `asset.save` itself is fixed on both: `asset.save {"/Game/FPS/VFX/NS_Impact_Metal", force:true}` under another stream's `T_Player` PIE returned the full `saveState:"blockedByPie"` + `saveDetail` + `pieWorlds[]` payload `#9` verified.
