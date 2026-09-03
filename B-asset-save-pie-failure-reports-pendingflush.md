---
id: B-asset-save-pie-failure-reports-pendingflush
title: "asset.save reports pendingFlush:true with no saveState when PIE blocks the write, so a hard failure reads as a retryable throttle"
status: OPEN
severity: High
category: bug
tags: [asset-save, pie, savestate, pendingflush, silent-false-signal, diagnostics, multi-agent]
encounters: 7
lastSeen: 2026-09-03T00:15:00Z
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

## History

- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8, shared editor, port 27145) authoring `/Game/FPS/VFX/Emitters/E_Explosion_Flash`. Emitter was fully configured and `niagara.compile {force:true, wait:true}` returned `status:"completed"`; two consecutive `asset.save {force:true}` calls each returned `saved:false, sizeBytes:0, pendingFlush:true` with no `saveState`, and `ls` confirmed no `.uasset` on disk. The editor log for the same calls records `LogUtils: Error: The Editor is currently in a play mode.` plus `SaveAssetToDiskReportingPresence ... state=failed outcome=Failed forced=true`, i.e. the cause was PIE held by another agent in the shared editor and the handler knew the state was `failed` while replying `pendingFlush`. Cross-ref `B-editor-save-all-pie-diagnostic` (DONE) — same defect class, fixed for `editor.save_all` only, and `asset.save` is the verb the docs steer callers to instead. Diagnosis from the response payloads, the log lines quoted above, and `safe-mutation-save.md`'s `saveState` table; no plugin source read.
- `#2-second-encounter-stale-sizebytes` `OPEN` reporter — Independently hit on the same editor/session authoring `/Game/FPS/VFX/Emitters/E_Smoke_Smoke` (smoke-grenade emitter stream). Adds one detail #1 could not show: the asset **already existed on disk**, so the two `asset.save {force:true}` replies were `{"saved":false,"sizeBytes":114702,"pendingFlush":true}` — `sizeBytes` is **non-zero and equal to the stale on-disk size**, not `0`. A caller who sanity-checks `sizeBytes > 0` (a natural guard, since #1's failure showed `sizeBytes:0`) is therefore misled twice over: the payload looks like a real write of a plausibly-sized package that is merely queued. `asset.is_dirty` on the same package returned `isDirty:true` immediately after, and the matching log lines are `SaveLoadedAssetThrottled: failed to save '/Game/FPS/VFX/Emitters/E_Smoke_Smoke.E_Smoke_Smoke'` + `SaveAssetToDiskReportingPresence ... state=failed outcome=Failed forced=true dirtyBefore=true existedBefore=true sizeBefore=114702 sizeAfter=114702 stampBefore=2026-09-02T19:54:26.000Z stampAfter=2026-09-02T19:54:26.000Z` (19:55:17 and 19:55:24 UTC). Suggests the fix should also stop reporting `sizeBytes` from the pre-existing file on a failed write, or label it `sizeBytesOnDisk`, so it cannot be read as the size just written.
- `#3-recurs-after-pie-restarts` `OPEN` reporter — Third occurrence, same session, and the one that shows the failure is environmental and time-varying rather than a property of the asset. `/Game/FPS/VFX/Emitters/E_Explosion_Light` **saved successfully** at 20:00:57Z (`saved:true, sizeBytes:88302`, `.uasset` mtime confirmed). 54 seconds later, after two further edits and a clean `niagara.compile status:"completed"`, the identical `asset.save {force:true}` on the identical asset returned `{"saved":false,"sizeBytes":88302,"pendingFlush":true}` — and the log again shows `LogUtils: Error: The Editor is currently in a play mode.` plus `SaveAssetToDiskReportingPresence ... state=failed outcome=Failed forced=true ... stampBefore=2026-09-02T20:00:57.000Z stampAfter=2026-09-02T20:00:57.000Z`. A different agent had re-entered PIE in the interim; `tail -200` of the log carried 61 `play mode` errors. Confirms `#2`'s `sizeBytes` point on a second asset: the reply's `sizeBytes` is the **unchanged pre-existing file's** size, and here it is byte-identical to the size the *successful* save reported a minute earlier, so comparing `sizeBytes` against a known-good previous save cannot detect the failure either. Practical note for whoever fixes this: because PIE is global and is entered and left repeatedly by unrelated agents, the same call alternates between success and silent-looking failure minute to minute, so any caller that does not read the log will produce assets that are durable or not essentially at random.

- `#4-fourth-stream-and-the-symmetric-case` `OPEN` reporter — Fourth independent stream hit this, and this sighting adds the *symmetric* case the first three do not cover: the blocked writer was not the agent running PIE, and the agent running PIE was not doing anything wrong. WEAPONS called `blueprint.scs.set_property` on `/Game/FPS/Weapons/BP_Weapon_AR` and `BP_Weapon_Pistol` (setting `StaticMesh` on the inherited `WeaponMesh` component); both returned `success: true, compiled: true, source: "inherited_scs"`. The following `asset.save {force: true}` on each returned `saved: false, pendingFlush: true` — **with a `sizeBytes` that had grown** (34835 -> 35756 and 36164 -> 37093), which reads as "the file was written and is 35756 bytes". It was not: `ls` gave an mtime of 23:09:55 local, minutes earlier, and `grep -ac SM_WPN` on both `.uasset` files returned **0** — the mesh assignment was not in either file. `editor.pie_status` then showed `inPie: true` with one standalone context on `/Game/FPS/Test/UEDPIE_0_T_UI.T_UI`, i.e. the **UI** stream's session, taken under the project's world-lock protocol while WEAPONS was doing lock-free asset work — which the protocol explicitly permits and encourages. So the reporting problem is worse than "your own PIE blocks your saves": there is no reason for the blocked agent to think about PIE at all, and nothing in the response mentions it. **`sizeBytes` on a `saved: false` response is the field that did the damage here** — it is the in-memory expectation, not what is on disk, and paired with `pendingFlush` it reads as a throttle that already half-succeeded. Two additions to this ticket's existing ask, both cheap: name the blocker (`saveState: "blockedByPie"` plus the PIE map, which `editor.pie_status` already knows) so the message is actionable by someone who did not start the session; and **omit or zero `sizeBytes` when `saved` is false**, because a byte count on a write that did not happen is a claim about disk that is not true. Disk proof used here was `ls` mtime plus `grep -a` on the bytes, per the project rule that landed today after `asset.reload` crashed the editor a third time — worth noting because reload is the check an agent would otherwise reach for and it is now forbidden.
- `#N-audio-stream-blueprint-migration` `OPEN` reporter — Hit again on the FPS AUDIO stream while migrating `/Game/FPS/Audio/BP_DA_ImpactSFX`'s two lookup maps from a `byte` key to `EPhysicalSurface`. `asset.save {assetPath, force:true}` answered `{saved:false, sizeBytes:16180, pendingFlush:true}` twice, with **no error, no `saveState`, and no mention of PIE**. `asset.is_dirty` confirmed `isDirty:true`. The file on disk was an hour stale (mtime 22:41:23 against a 23:49 edit) and `grep -a EPhysicalSurface` on the `.uasset` returned **0**, so the whole type migration existed only in memory. Only `editor.save_all` diagnosed it, and precisely: `SAVE_FAILED ... (PIE active; 3 asset(s) locked by PIE)` with `failedAssets:[{path:.../BP_DA_ImpactSFX, reason:"BlockedByPie"}]` — another stream (WEAPONS) was running a PIE fire/reload verification and PIE holds Blueprint packages. Two things make this expensive rather than cosmetic. First, `pendingFlush:true` is the *same* answer `asset.save` gives for the benign 0.5 s throttle, so the documented remedy (retry with `force:true`) is exactly wrong here and I burned two retries on it. Second, in a shared editor the blocker belongs to a different agent, so the caller cannot even guess the cause from its own actions — `save_all` knows the reason and `asset.save` discards it. Ask: surface the same `BlockedByPie` reason (and a `saveState` of `failed`, not a bare `pendingFlush`) from `asset.save`. Workaround: when `asset.save` reports `pendingFlush` twice on an unchanged mtime, call `editor.save_all` purely to read the reason, then wait out the other stream's PIE session.
- `#N-audio-synth-export-batch-nonpie-variant` `OPEN` reporter — Hit on the FPS AUDIO stream re-synthesising the 12 `SW_Step_*` footsteps and 5 ambience/foley waves. Sixteen consecutive `audio.synth.export {save:true}` calls returned `verification.pass:true, existsAfter:true, existsOnDisk:true` alongside `saved:false, pendingFlush:true, pendingSave:true`; six follow-up `asset.save {assetPath, force:true}` calls each answered `{saved:false, sizeBytes:<stale on-disk size>, pendingFlush:true}` with **no `saveState`**. `ls` confirmed all 11 files still carried their pre-edit mtime (22:43) more than an hour later, reproducing `#2`/`#4`'s point that `sizeBytes` on a `saved:false` reply is the stale file, not a write. The new datum is the escape hatch: `unreal.EditorLoadingAndSavingUtils.save_packages(pkgs, False)` run through `python.execute` wrote **all 12 packages in one call** and returned `True`, and `ls` then showed every mtime advanced — so on this occasion a different save API succeeded on the same packages within seconds of `asset.save {force:true}` refusing them. I did not establish whether another stream's PIE was up at the moment of either call, so this is not offered as a counter-example to the PIE root cause; what it does show is that `asset.save`'s `pendingFlush` was not describing a condition that blocked *all* writers, and that a caller with no `saveState` to read cannot tell the two situations apart. Reinforces this ticket's existing ask (`saveState` + a named blocker, and drop `sizeBytes` when `saved:false`), and adds: whatever `asset.save {force:true}` is doing, it is weaker than `save_packages(..., only_dirty=False)`, so either it should route through that on `force`, or the docs should name `save_packages` as the real forced path. Disk proof was `ls` mtime plus `grep -a` on the `.uasset` bytes throughout; `asset.reload` was not used.
- `#N-pie-root-cause-confirmed-from-log-audio-waves` `OPEN` reporter — Closes the gap `#N-audio-synth-export-batch-nonpie-variant` names explicitly ("I did not establish whether another stream's PIE was up"). Same session, FPS AUDIO stream, re-synthesising the ten weapon/impact waves. `audio.synth.export {save:true}` on `SW_Impact_Glass_A` returned `verification.pass:true` beside `saved:false, pendingFlush:true`; two `asset.save {force:true}` retries answered the same with the **stale** `sizeBytes:101173`. The editor log carries the cause on the adjacent line and the RPC does not: `LogUtils: Error: The Editor is currently in a play mode.` immediately precedes each `SaveLoadedAssetThrottled` / `SaveAssetToDiskReportingPresence: ... state=failed outcome=Failed forced=true`, and `LogPlayLevel: Creating play world package: /Game/FPS/Test/UEDPIE_0_T_Weapons` bounds the window. **The same candidate exported and saved cleanly on the first attempt after `LogWorld: BeginTearingDown`, with no recipe change** — so on this occasion PIE *is* demonstrably the blocker, and the identical symptom recurred minutes later inside a second PIE window (`UEDPIE_0_T_AI`) across seven different packages, all of which then saved `true` on one retry once that PIE ended. Two consequences for the ask: (a) the blocker is already known to the handler at the moment it answers — `LogUtils` prints it one line earlier — so `saveState:"failed"` plus a named `saveBlocker:"pie"` is a plumbing change, not new detection work; (b) since the block is a *window*, not a property of the asset, the useful response field is "retry when PIE ends", which is precisely what today's `pendingFlush:true` with no `saveState` cannot distinguish from the 0.5 s throttle it looks identical to. Disk proof throughout was `ls`/`stat` mtime plus `grep -a` on the `.uasset` bytes; `asset.reload` was not used. Cost here was ~15 min of blocked exports plus one wrong initial diagnosis (read as a throttle, retried twice for nothing).
